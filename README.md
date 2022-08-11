# Bài tiểu luận: Mô tả hoạt động về một hệ thống phân tán data dạng timeseries

Video thực hành: https://drive.google.com/file/d/1sG5DpmAWHDK6vGraLa0nuiAK0wm6jGFB/view?usp=sharing

## Lý thuyết
Phân tán dữ liệu (clustering) là một giải pháp chia nhỏ một Database lớn thành nhiều Database nhỏ, 
ta có thể phân tách từng bảng (partition) hoặc cả một DB ra nhiều phần nhỏ đặt ở nhiều máy chủ (server) khác nhau (sharding). 
Điều này sẽ giúp cho hệ thống DB của chúng ta đạt được các tính chất khả năng bảo trì (manageability), hiệu xuất (performance), 
tính sẵn sàng (availability), và cân bằng tải (load balancing). 
Và giải pháp này cũng giảm chi phí cũng như tính mở rộng (scalability) để scale up DB bằng cách dùng nhiều server nhỏ gộp lại hơn 
là nâng cấp một server lớn.

Replication – Sao chép toàn bộ table hoặc database vào nhiều server khác nhau. Có thể dễ hiểu hơn là các node server có data giống 
hệt nhau và được đồng bộ liên tục để đảm bảo dữ liệu luôn luôn giống nhau giữa các node. Được sử dụng để cải thiện tốc độ truy cập vào
các bản ghi tham chiếu như dữ liệu chính.

Partitioning – Chia nhỏ một cơ sở dữ liệu nguyên khối lớn thành nhiều cơ sở dữ liệu nhỏ hơn dựa trên sự gắn kết dữ liệu. 
Ví dụ – Chia nhỏ một table lớn theo ngày, tháng hoặc năm, theo category …

Clustering – Từ góc độ cơ sở dữ liệu, phân cụm là khi bạn có một nhóm node (server) lưu trữ cùng một database schema trên cùng một phần mềm CSDL 
với một số hình thức trao đổi dữ liệu giữa các server này. Từ bên ngoài cụm, các server này được xem như một đơn vị duy nhất chứa một liên minh dữ liệu 
được trải rộng trên các node trong cụm. Khi ứng dụng của bạn truy cập vào một cụm, yêu cầu cuối cùng được chuyển đến một node duy nhất trong cụm để đọc 
hoặc ghi hoạt động.

Sharding – Chia nhỏ một bảng dữ liệu lớn theo chiều ngang . Một bảng chứa 100 triệu hàng có thể được chia thành nhiều bảng chứa 1 triệu hàng mỗi bảng. 
Mỗi bảng do sự phân chia sẽ được đặt vào một cơ sở dữ liệu / server riêng biệt. Sharding được thực hiện để phân tán tải và cải thiện tốc độ truy cập. 
Facebook /Twitter đang sử dụng kiến trúc này.

Replication, Partitioning, Sharding là một hình thức của clustering trong đó tất cả các node trong cluster c
ó schema và data giống nhau / giống hệt nhau/ được chia nhỏ và phân tán. 

## Thực hành

Tạo 1 database Postgresql có tính năng partition áp dụng cho data dạng timeseries

Vì sao là Postgresql thay vì SQL Server:
- Postgres mã nguồn mở và có nhiều extension hỗ trợ việc partition, sharding, clustering
- Postgres hỗ trợ cả Linux và Window, SQL Server thuộc sở hữu của Microsoft nên kém friendly hơn voi các hệ điều hành khác
- Postgres, docker là những công nghệ đang hot và được áp dụng nhiều ở industry

Vì sao là partition:
- Giống như Replication và Sharding, Partitioning cũng là một loại CSDL phân tán 
- Partition giúp giảm thời gian truy vấn dữ liệu. Thực hiện hiện partition trong một số trường hợp giúp ta optimizing database.
- Partition là lựa chọn hoàn hảo cho CSDL IoT, ..

Tiếp theo sẽ hướng dẫn tạo 1 hypertable, tự động partitioned theo “time” dimension
### In project directory

docker-compose up -d

Access pgadmin: localhost:8880 with user and database server in .env 

### in table query tool

CREATE TABLE sensors(
  id SERIAL PRIMARY KEY,
  type VARCHAR(50),
  location VARCHAR(50)
);

CREATE TABLE sensor_data (
  time TIMESTAMPTZ NOT NULL,
  sensor_id INTEGER,
  temperature DOUBLE PRECISION,
  cpu DOUBLE PRECISION,
  FOREIGN KEY (sensor_id) REFERENCES sensors (id)
);

#### transform the sensor_data table into a “Hypertable” partitioned on the “time” dimension

SELECT create_hypertable('sensor_data', 'time');

#### create a special index on the sensor ID, since we’re very likely to filter on both sensor ID and time.

create index on sensor_data (sensor_id, time desc);

### in sensor

#### add a few different sensors

INSERT INTO sensors (type, location) VALUES
  ('a','floor'),
  ('a', 'ceiling'),
  ('b','floor'),
  ('b', 'ceiling');

### in sensor data

#### create some simulated time series data

INSERT INTO sensor_data 
  (time, sensor_id, cpu, temperature)
SELECT
  time,
  sensor_id,
  random() AS cpu,
  random()*100 AS temperature
FROM 
  generate_series(
    now() - interval '31 days', 
    now(), interval '5 minute'
  ) AS g1(time), 
  generate_series(1,4,1) AS g2(sensor_id);

#### Run a simple select query to see some of our newly-simulated data

SELECT * 
FROM sensor_data
WHERE time > (now() - interval '1 day')
ORDER BY time;

SELECT 
  time_bucket('30 minutes', time) AS period, 
  AVG(temperature) AS avg_temp, 
  last(temperature, time) AS last_temp --the latest value
FROM sensor_data 
GROUP BY period;

SELECT 
  t2.location, --from the second metadata table
  time_bucket('30 minutes', time) AS period, 
  AVG(temperature) AS avg_temp, 
  last(temperature, time) AS last_temp, 
  AVG(cpu) AS avg_cpu 
FROM sensor_data t1 
INNER JOIN sensors t2 
  on t1.sensor_id = t2.id
GROUP BY 
  period, 
  t2.location;

CREATE VIEW sensor_data_1_hour_view
WITH (timescaledb.continuous) AS --TimescaleDB continuous aggregate
SELECT 
  sensor_id,
  time_bucket('01:00:00'::interval, sensor_data.time) AS time,
  AVG(temperature) AS avg_temp, 
  AVG(cpu) AS avg_cpu
FROM sensor_data
GROUP BY 
  sensor_id,
  time_bucket('01:00:00'::interval, sensor_data.time)

# Reference
https://mccarthysean.dev/timescale-dash-flask-part-1#fromHistory#fromHistory