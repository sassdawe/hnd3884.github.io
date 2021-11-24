---
title: Install and debug Confluence
author: HoangND
date: 2021-11-25 00:00:00 +0700
categories: [Tutorial]
tags: [java, tutorial]
---

Hướng dẫn cài đặt và debug Confluence. Lần đầu debug java application lưu lại có gì sau còn dễ kiếm 🤭

## Install confluence
#### PostgreSQL
Hướng dẫn cài đặt PostgreSQL tại [đây](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)

Setup database cho confluence
```bash
# Change password for postgres user
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'hoang123';
# Create database
sudo -s -u postgres
createdb --owner postgres --encoding utf8 confluence
```


#### Confluence
Vào trang chủ confluence tại [đây](https://www.atlassian.com/software/confluence/download-archives) để download file cài đặt

Vào thư mục chứa file cài đặt

```bash
# add excute permission
sudo chmod +x atlassian-confluence-7.13.2-x64.bin
# install confluence
sudo ./atlassian-confluence-7.13.2-x64.bin
```

Mặc định các file jar của Confluence sẽ nằm trong thư mục /opt/atlassian/confluence và dữ liệu nằm trong /var/atlassian/application-data/confluence. Port mặc định của confluence là 8090.
Vào http://localhost:8090/ bằng browser để tiếp tục cài đặt.

![image](https://user-images.githubusercontent.com/61985236/143288957-b0349eec-b8df-417f-a6a0-72c081eb9995.png)
_Database connection_

## Debug confluence
#### Set up confluence
Chỉnh sửa file /opt/atlassian/confluence/bin/setenv.sh, thêm một dòng
```
START_CONFLUENCE_JAVA_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
```

![image](https://user-images.githubusercontent.com/61985236/143294309-230aa768-56f1-4322-b1ec-5db47e3b999e.png)

#### Debug
Mở thư mục /opt/atlassian/confluence/confluence bằng IntelliJ IDEA. 

![image](https://user-images.githubusercontent.com/61985236/143294870-f7f1f765-1a6d-428d-b104-89473ec5ab2d.png)

Chuột phải vào project -> Open Module Setting -> Project Setting -> Libraries -> + -> Tìm đến /opt/atlassian/confluence/confluence/WEB-INF/lib/confluence-xxx.jar

![image](https://user-images.githubusercontent.com/61985236/143295169-d52756b6-8e77-4494-bdee-d0ed51bba8ea.png)

Ví dụ debug chức năng đăng nhập, tìm kiếm tất cả bằng cách ấn 2 lần Shift

![image](https://user-images.githubusercontent.com/61985236/143296049-075855f3-358c-408a-9633-566a11e6537f.png)

Đặt debug đâu đó trong các hàm Login. Chọn Debug -> Edit configuration -> New -> Remote JVM Debug

![image](https://user-images.githubusercontent.com/61985236/143296302-ef15e1dc-79d5-4692-b07e-7aabe9a585d7.png)

Debug -> Run 'debug confluence'. Vào confluence và đăng nhập

![image](https://user-images.githubusercontent.com/61985236/143297672-e67511b9-c921-4e7f-9fee-d640fe24a15f.png)



