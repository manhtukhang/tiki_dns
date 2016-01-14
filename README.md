# Triển khai DNS Server sử Docker và RancherOS

## Mục lục
- [Giới thiệu](#giới-thiệu)
- [Cài đặt](#cài-đặt)
  + [RancherOS](#rancheros)
  + [Bind](#bind)
- [Sử dụng](#sử-dụng)
  + [Bind](#bind)
  + [Các thao tác cơ bản](#các-thao-tác-cơ-bản)


# Giới thiệu
  - [Docker](https://www.docker.com/what-docker) là một giải pháp ảo hóa mã nguồn mở có thể đóng gói và chạy bất kỳ phần mềm nào dưới dạng một *container* gọn nhẹ. Thử tưởng tượng, với mỗi lần cài đặt phần mềm nào đó lại có rất nhiều gói phụ thuộc, mỗi hệ điều hành lại có những gói với tên và phiên bản khác nhau khiến cho thao tác cài đặt, bảo trì hết sức phức tạp và nhàm chán. Docker đóng gói tất cả vào 1 image và có thể đem đến bất cứ đâu để thực thi lại mà không cần quan tâm đó là hệ điều hành nào (miễn hỗ trợ Docker là được - tất nhiên Windows Server 2016 cũng đã hỗ trợ rồi).  
  - [RancherOS](http://rancher.com/rancher-os/) là một hệ điều hành tối giản hết mức để chạy Docker, dung lượng chỉ hơn 25Mb và chiếm khoảng 100Mb RAM khi chạy.
  - `Bind` là phần mềm mã nguồn mở dùng để xây dựng DNS server với ưu điểm dễ cấu hình, chiếm ít tài nguyên hệ thống.

# Cài đặt

## RancherOS
- Tải file iso phiên bản mới nhất tại [đây](https://github.com/rancher/os#latest-release)  
- Quá trình cài đặt và cấu hình `RancherOS` được tự động qua file cấu hình `cloud-config.yaml` có dạng như sau:  

```yml
  #cloud-config
  
  ssh_authorized_keys:
    - ssh-rsa AAA...fZQ==it@tiki.vn
    - ssh-rsa AAA...sGQ==it.helpdesk@tiki.vn
  
  rancher:
    network:
      interfaces:
        eth0:
          address: x.x.x.x/24
          gateway: x.x.x.x
          mtu: 1500
          dhcp: false
      dns:
        nameservers:
          - x.x.x.x
        domain:
          - tiki.com.vn
  hostname: xxx
```
- Trong đó:
  - Dòng đầu tiên `#cloud-config` bắt buộc phải có để nhận diện đây là file cấu hình tự động  
  - `ssh_authorized_keys` là các khóa ssh công cộng (ssh public key) của máy tính cần truy cậo vào server, mỗi admin cần 1 khóa, **không** nên dùng chung  
  - `address` là địa chỉ IP tĩnh cần đặt, ví dụ *x.x.x.12*  
  - `gateway` là địa chỉ IP của *defaut gateway*, ví dụ *x.x.x.1*  
  - `nameservers` là địa chỉ IP của các DNS có sẵn, ví dụ *x.x.x.20*  
  - `hostname` là tên của server này, ví dụ *xxxdnsyyy*  
 
- Khi đã có file `cloud-config`, tiến hành boot từ file iso mới nhất ở trên và gõ lệnh sau để cài đặt
```bash
    sudo ros install -c cloud-config.yaml -d /dev/sda -f
```
với `/dev/sda` là tên ổ đĩa.

- Sau khi cài đặt hoàn tất sẽ có thông báo yêu cầu reboot, quá trình cài đặt và cấu hình `RancherOS` hoàn tất, hết sức đơn giản  

## Bind

- Quá trình cài đặt `bind` không tiến hành theo cách thông thường mà sẽ đóng gói tất cả vào 1 image bằng Docker  
- Docker phiên bản (gần như) mới nhất đã được tích hợp vào `RancherOS` nên không cần cài đặt nữa  
- Ở đây, chúng ta chỉ việc build 1 *container (image)* chứa tất cả những gì cần thiết cho DNS server thông qua file `Dockerfile`  

```dockerfile
FROM        ubuntu:trusty
MAINTAINER  tu.khang@tiki.vn

ENV RECORD_DIR=/var/named/records \
    PID_DIR=/var/run/named

RUN echo 'APT::Install-Recommends 0;' >> /etc/apt/apt.conf.d/01norecommends \
 && echo 'APT::Install-Suggests 0;' >> /etc/apt/apt.conf.d/01norecommends \
 && apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y nano net-tools ca-certificates bind9 dnsutils dnstop \
 && rm -rf /var/lib/apt/lists/*

RUN mkdir -p ${RECORD_DIR} \
 && mkdir -p ${PID_DIR}} \
 && chown -R bind:bind ${RECORD_DIR} \
 && chown -R bind:bind ${PID_DIR}}

EXPOSE 53/udp
VOLUME ["${RECORD_DIR}"]

ENTRYPOINT ["/usr/sbin/named", "-g", "-c", "/etc/bind/named.conf", "-u", "bind"]

```
- Gõ lệnh sau để build *image*

```bash
  docker build -t [tên image]:[tag] [đường dẫn thư mục chứa Dockerfile]
```
Chúng ta có thể tạo ra nhiều phiên bản với các *tag* khác nhau  

Ví dụ:

```
  docker build -t xxdnsxxx:core ./build
```


# Sử dụng

## Bind

- Việc cấu hình để `bind` trở thành DNS Server hết sức đơn giản, ở đây chúng ta chạy ở chế độ `Slave` và cần lấy các `record` từ `Master` Server  
- Chúng ta cần 1 file chứa thông tin các `DNS zone` với tên `named.conf.local` như sau:
```
  // Forward zones
  
  zone "tiki.com.vn" IN {
    type slave;
    masters {
      x.x.x.x;
    };
    file "/var/named/records/db.tiki.com.vn";
  };
  
  zone "_msdcs.tiki.com.vn" IN {
    type slave;
    masters {
      x.x.x.x;
    };
    file "/var/named/records/db.msdcs.tiki.com.vn";
  };
  
  // Revert zones
  
  zone "x.y.z.in-addr.arpa" IN {
    type slave;
    masters {
      x.x.x.x;
    };
    file "/var/named/records/db.x.y.z";
  };
  
  ...
  
  include "/etc/bind/rndc.key";

```

- Trong đó:
  - `zone "..."` là tên các zone, bao gồm `Forward zones` và `Revert zones`  
  - `x.x.x.x` là *master server*, nơi có thể thay đổi các record DNS  

- Tiếp theo là các opption cho DNS server, nằm trong file `named.conf.options`:

```
  options {
    directory "/var/cache/bind";
    allow-recursion { x.x.0.0/16;
                      y.y.y.0/24;
    };
    dnssec-validation no;
    auth-nxdomain no;    # conform to RFC1035
    listen-on-v6 { any; };
};

```

- Lưu ý dòng `allow-recursion` là các *Network* được phép query DNS  

## Các thao tác cơ bản

- Tạo và khởi chạy `container` chứa DNS Server
```
  docker run -d --restart="always" \
                --name="[tên container]" \
                --hostname="[hostname của container]" \
                --volume [đường dẫn chứa file named.conf.options]:/etc/bind/named.conf.options \
                --volume [đường dẫn chứa file named.conf.local]:/etc/bind/named.conf.local \
                --volume [đường dẫn thư mục để cache các record]
                --publish 53:53/udp \
                [tên image]
                
```
Ví dụ:
```
  docker run -d --restart="always" \
                --name="tkdnsxxx" \
                --hostname="tkdnsxxx" \
                --volume `pwd`/config/named.conf.options:/etc/bind/named.conf.options \
                --volume `pwd`named.conf.local:/etc/bind/named.conf.local \
                --volume `pwd`/records \
                --publish 53:53/udp \
                tkdnsxxx:core
                
```

- Dừng `container` chứa DNS Server
```
  docker stop [tên container]
```

- Chạy lại 1 container đã bị dừng
```
  docker start [tên container]
```

- Khởi động lại `container` chứa DNS Server
```
  docker restart [tên container]
```

- Theo dõi *real-time* các query
```
  docker exec -it [tên container] dnstop -l 3 eth0
```
Sau đó bấm các phím `1` `2` `3` `4` hoặc `t` để thay đổi chế độ xem, `Ctrl` + `C` để thoát  

- Các lệnh khác:
  + `docker ps` danh sách các container đang chạy  
  + `docker ps -a` danh sách tất cả các container  
  + `docker rm [tên container]` xóa 1 container theo tên  
  + `free -m` theo dõi thông tin RAM của server  
  + `top` theo dõi thông tin CPU, RAM, Proccess,... của server  
  
