# OpenStack 기본 구성

---

## 오픈스택 버전

- 2024.01 - caracal

## 서버 구성

| **구분** | **서버** | **IP** | **Controller Node** | **Compute Node** | storage Node |
| --- | --- | --- | --- | --- | --- |
| **운영** | mncsvrt06 | `192.168.73.171` | O | O | O |
|  | mncsvrt07 | `192.168.73.172` | X | O | O |
|  | mncsvrt08 | `192.168.73.173` | X | O | O |
|  | mncsvrt03 | `192.168.73.168` | X | O | O |
|  | mncsvrt04 | `192.168.73.169` | X | O | O |
|  | mncsvrt12 | `192.168.73.177` | X | O | O |
|  | mncstorage01 | `192.168.73.162` | X | X | O |
| **개발** | mncsvrt02 | `192.168.73.167` | O | O | O |

---

## 스토리지 구성

| **구분** | **서버** | **Block Storage** | **Object Storage** |
| --- | --- | --- | --- |
| **운영** | mncsvrt06 | 800GB / (dev/sdb1) | 300GB (dev/sdb2) |
|  | mncsvrt07 | 1.1T / (dev/sdb1) |  |
|  | mncsvrt08 | 800G / (dev/sda1) | 300GB (dev/sda2) |
|  | mncsvrt03 | 1.1T / (/dev/sdb1) |  |
|  | mncsvrt04 | 1.1T / (/dev/sdb1) |  |
|  | mncsvrt12 | 1.1T / (/dev/sdb1) |  |
|  | mncstorage01 | 7.2T / (/dev/sdb4) |  |
| **개발** | mncsvrt02 | 900GB / (dev/sdb1) | 100GB(dev/sdb2)
100GB(dev/sdb3) |

> 현재 별도의 Storage Node 없이 **Controller Node**와 **Compute Node**에 **Cinder**와 **Swift**를 구성함.
> 

---
