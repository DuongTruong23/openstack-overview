# Cinder:
**Cinder** là 1 module có dịch vụ cung cấp block storage trong OpenStack. Nó được thiết kế để thể hiện tài nguyên lưu trữ đã sử dụng tới người dùng bởi OpenStack compute.
- Persitent block storage được gắn vào compute instances/VMs. 
- Quản lý vòng đời Block Storage: create, delete, extend or attach or detach,...
- Mối quan hệ giữa instances and block storage device: Sau khi cắm vào drive mới, có thể format nó thành NTFS, ext3 or whatever you need.
- Quản lý snapshoots và các loại volumes
- Hoàn toàn tích hợp vào Nova và Horizon

## Cinder Architecture:

<img width="1384" height="797" alt="image" src="https://github.com/user-attachments/assets/cac9ec96-099d-4723-80eb-a683eb73acbf" />


- **Cinder-api:** là 1 ứng dụng wsgi, nó cho phép và xác thực thông qua rest API dựa trên yêu cầu được viết bằng file Json hoặc XML từ người dùng và truyền chúng đến các cinder processes khác thông qua message bus (RabbitMQ/AMQP). Các tiền trình có thể cử lý thông qua API:
    - Volume create/delete/list/show
    - Create from volume, image or snapshot
    - Snapshot create/delete/list/show
    - Volume attach/detach
    - support other operations related to volume types, quotas or backups
- **Cinder-scheduler:** sử dụng plugins có cơ chế filters và weights để lựa chọn backend storage tốt nhất dựa trên các tiêu chí như dung lượng khả dụng, hiệu suất.
    - Key Filters: CapacityFilter (kiểm tra free space), CapabilityFilter ( matches backend capabilities),...
- **Cinder-volume:** quản lý thiết bị block storage và điều phối tới storage backend. Thực hiện tiến trình như: create, delete, attach, detach,... volumes, snapshots,... bằng cách tương tác với storage backend đã được cấu hình thông qua giao thức Fibre Channel.
- **Cinder-backup:** Xử lý quá trình backup và sao lưu của volume. Tạo backups volumes vào hệ thống backup storage (cụ thể là CEPH) và khôi phục chúng khi cần.
- **Storage Backend Drivers:** gồm backend code cụ thể để giao tiếp với nhiều loại hệ thống storage. Eg: CEPH
    - Có thể chạy nhiều cinder-volume instances, mỗi driver có file cấu hình riêng về settings và storage backend.
    - 1 cinder-volume instance có thể quản lý nhiều backends. Mỗi backend driver được cấu hình để tương tác với 1 nhóm lưu trữ *
- **Database**: lưu trữ metadata về volumes, snapshots, backups và trạng thái của chúng. Nhằm duy trì bản ghi liên tục về tài nguyên và cấu hình của chúng.

## Cinder Workflow

1. Volume Creation:
- Gửi request tạo 1 volume qua Cinder-API (e.g., dùng Horizon hoặc CLI).
- API xác thực request và truyền nó đến message queue.
- Cinder-Scheduler nhận request, đánh giá và lựa chọn storage backends tốt nhất bằng cách sử dụng filters và weighers.
- Scheduler gửi request đến Cinder-Volume service và chạy trên storage backend đã được chọn.
- Cinder-Volume sử dụng storage driver phù hợp để tạo volume và updates database về metadata của volume.
- API trả kết quả về user.


## Example of Data and Control Traffic For Cinder / CInder Attach Workflow

- Cinder tương tác trực tiếp với Nova để cung cấp persistent block storage volumes, snapshots và backups tới instances nơi mà Nova quản lý bằng cách call API.
- Let's say an instance requires a volume. This means Nova and Cinder should talk through their rest APIs. After getting their request via its API, cinder builds and returns all of the info needed by Nova to attach the specified volume to the instance.
- Nova does a volume attachment and both services update their databases with the latest status.
- The important point I want to mention is the fact that the data path is over an iSCSI connection over the network data path and control path are completely different.

<img width="1176" height="891" alt="image" src="https://github.com/user-attachments/assets/879c4fff-289a-4129-989f-0e71260578c4" />

## Cinder Status

|Status|Mô tả|
|------|-----|
|Creating|Volume được tạo ra|
|Available|Volume ở trạng thái sẵn sàng để attach vào một instane|
|Attaching|Volume đang được gắn vào một instane|
|In-use|Volume đã được gắn thành công vào instane|
|Deleting|Volume đã được xóa thành công|
|Error|Đã xảy ra lỗi khi tạo Volume|
|Error deleting|Xảy ra lỗi khi xóa Volume|
|Backing-up|Volume đang được back up|
|Restore_backup|Trạng thái đang restore lại trạng thái khi back up|
|Error_restoring|Có lỗi xảy ra trong quá trình restore|
|Error_extending|Có lỗi xảy ra khi mở rộng Volume|


## Managing Cinder from CLI
- cinder service-list
- cinder service-disable
- cinder service-enable
- openstack command list | grep openstack.volume -A 40: see all cinder commands
- openstack volume create --size 1 vol 1: create volume with specify size (1BG name vol1)
- openstack volume list: list volumes
- openstack server list: list servers
- openstack server add volume demo1 vol1: server add volume vol1 to demo1 server

- openstack backup create --name backup1 --force vol1
- openstack volume backup show backup1

- openstack snapshot create --name snap1 --force vol1

** Others:
- sudo mkfs.etx3 /dev/vdc: format ....
- sudo mkdir /mydisk
- sudo mount /dev/vdc /mydisk
---> volume has been created and attched to an instances.

### Volumes
- Persistent R/W Block storage
- Attached to instances as secondary storage
- Can be used as root volume to boot instances
- Volume lifecycle management
    - Create, delete, extend volumes
    - Attach/detach volumes
- Manages volume management

### Snapshoot
- A read only copy of a volume
- Create/delete snapshoots
- Create a volume out of a snapshot

### Backup
- Backup is an admin operation
- Done from CLI
- Backup stored in Swift: Needs a container
- Create/Restore backups
- $ cinder backup-create "volume-id"

### Quotas
- Quota can be set for operations limits
- Enforced per project:
    - Number of volumes
    - Storage space in GB
    - Number of snapshots


# Vocabulary for Cinder concept:
- Logical Volumes Manager (LVM)
- iSCSI Qualified Name (IQN): là format định dạng phổ biến cho tên iSCSI, dùng để xác định duy nhất các nút trong mạng iSCSI
    - Format: iqn.yyyy-mm.domain:identifier
    - Ex: iqn.2015-10.org.openstack.408ae959bce1
- NVMe Qualified Name (NQN): là format định dạng phổ biến cho tên NVMe, xác định duy nhất các máy chủ hoặc hệ thống con NVM trong mạng. Có 2 formats:
    - 1st format: nqn.yyyy-mm.domain:identifier
      Ex: nqn.2014-08.com.example:nvme:nvm-subsystem-sn-d78432
    - 2nd format: nqn.2014-08.org.nvmexpress:uuid:identifier
      Ex: nqn.2014-08.org.nvmexpress:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6