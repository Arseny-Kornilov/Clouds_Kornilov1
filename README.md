# Домашнее задание к занятию «Организация сети»
## Задание 1. Yandex Cloud
### Создать пустую VPC. Выбрать зону.
### Публичная подсеть.
### - Создать в VPC subnet с названием public, сетью 192.168.10.0/24.
```console
Public subnet
resource "yandex_vpc_subnet" "public" {
  name           = "public"
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.network-netology.id
  v4_cidr_blocks = ["192.168.10.0/24"]
}
```
### - Создать в этой подсети NAT-инстанс, присвоив ему адрес 192.168.10.254. В качестве image_id использовать fd80mrhj8fl2oe87o4e1.
```console
resource "yandex_compute_instance" "nat-instance" {
  name = "nat-instance"
  hostname = "nat-instance"
  zone     = var.yc_zone
  resources {
    cores  = 2
    memory = 2
  }
  boot_disk {
    initialize_params {
      image_id = "fd80mrhj8fl2oe87o4e1"
    }
  }
  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    ip_address = "192.168.10.254"
    nat       = true
  }
  metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}
```
### - Создать в этой публичной подсети виртуалку с публичным IP, подключиться к ней и убедиться, что есть доступ к интернету.
```console
resource "yandex_compute_instance" "public-instance" {
  name = "public-instance"
  hostname = "public-instance"
  zone     = var.yc_zone
  resources {
    cores  = 2
    memory = 2
  }
  boot_disk {
    initialize_params {
      image_id = "fd826honb8s0i1jtt6cg"
    }
  }
  network_interface {
    subnet_id = yandex_vpc_subnet.public.id
    nat       = true
  }
  metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}
```

```console
ssh ubuntu@51.250.72.109

ubuntu@public-instance:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=58 time=21.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=58 time=21.0 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=58 time=21.1 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=58 time=21.0 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 21.012/21.183/21.648/0.269 ms
```
### Приватная подсеть.
### - Создать в VPC subnet с названием private, сетью 192.168.20.0/24.
```console
resource "yandex_vpc_subnet" "private" {
  name           = "private"
  zone           = var.yc_zone
  network_id     = yandex_vpc_network.network-netology.id
  route_table_id = yandex_vpc_route_table.netology-routing.id
  v4_cidr_blocks = ["192.168.20.0/24"]
}
```
### - Создать route table. Добавить статический маршрут, направляющий весь исходящий трафик private сети в NAT-инстанс.
```console
resource "yandex_vpc_route_table" "netology-routing" {
  name       = "netology-routing"
  network_id = yandex_vpc_network.network-netology.id
  static_route {
    destination_prefix = "0.0.0.0/0"
    next_hop_address   = "192.168.10.254"
  }
}
```
### - Создать в этой приватной подсети виртуалку с внутренним IP, подключиться к ней через виртуалку, созданную ранее, и убедиться, что есть доступ к интернету.
```console
resource "yandex_compute_instance" "private-instance" {
  name = "private-instance"
  hostname = "private-instance"
  zone     = var.yc_zone
  resources {
    cores  = 2
    memory = 2
  }
  boot_disk {
    initialize_params {
      image_id = "fd8bkgba66kkf9eenpkb"
    }
  }
  network_interface {
    subnet_id = yandex_vpc_subnet.private.id
  }
  metadata = {
    ssh-keys = "ubuntu:${file("~/.ssh/id_ed25519.pub")}"
  }
}
```

```console
  yc compute instance list
+----------------------+------------------+---------------+---------+----------------+----------------+
|          ID          |       NAME       |    ZONE ID    | STATUS  |  EXTERNAL IP   |  INTERNAL IP   |
+----------------------+------------------+---------------+---------+----------------+----------------+
| xxxxxxxxxxxxxxxxxxxx | public-instance  | ru-central1-a | RUNNING | 51.250.1.121   | 192.168.10.3   |
| xxxxxxxxxxxxxxxxxxxx | private-instance | ru-central1-a | RUNNING |                | 192.168.20.19  |
| xxxxxxxxxxxxxxxxxxxx | nat-instance     | ru-central1-a | RUNNING | 84.201.134.247 | 192.168.10.254 |
+----------------------+------------------+---------------+---------+----------------+----------------+
```

```console
scp ~/.ssh/id_ed25519 ubuntu@51.250.1.121:/home/ubuntu/.ssh/
id_ed25519    

ssh ubuntu@51.250.1.121

ssh 192.168.20.19

ubuntu@private-instance:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=54 time=17.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=54 time=16.3 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=54 time=16.2 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 16.171/16.610/17.358/0.531 ms
```
