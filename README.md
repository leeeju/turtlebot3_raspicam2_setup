# turtlebot3_raspicam2_setup
터틀봇3에 raspicam2 노드 설치하기

**참조:** 
- https://jdu-stuff.tistory.com/50
- https://askubuntu.com/questions/1130052/enable-i2c-on-raspberry-pi-ubuntu
- [https://github.com/christianrauch/raspicam2_node.git](https://github.com/christianrauch/raspicam2_node.git)

## 구성환경
- RaspberryPi 4
- Ubuntu Server 20.04
- R.O.S : foxy

1. 터틀봇3의 SBC(RaspberryPi)에 ssh 연결
2. 라즈베리파이와 ssh 연결을 위해 `ssh ubuntu@<라즈베리파이의 IP 주소>` 를 실행한다. 
```bash
ssh ubuntu@<라즈베리파이의 IP 주소>
```
패스워드( turtlebot )를 입력한다. 
```bash
ubuntu@<라즈베리파이의 IP 주소>'s password: 
```
## 소스코드 가져오기
작업폴더를 자신이 사용하는 ROS 2 워크스페이스( `~/turtlebot3_ws/src` 같은 )로 변경.
터틀봇3에는 2개의 ROS 2 워크스페이스가 존재한다. `~/turtlebot3_ws` 와 `~/colcon_ws` 이다. `~/turtlebot3_ws` 를 사용하는 경우에는 상관 없지만, `~/colcon_ws` 를 사용하는 경우에는 반드시 `~/.bashrc` 파일에 `source ~/colcon_ws/install/local_setup.bash` 를 추가한다.
```bash
cd ~/colcon_ws/src
```
`git clone` 으로 `raspicam` 노드의 소스코드를 ROS 2 워크스페이스로 복사
```bash
git clone https://github.com/christianrauch/raspicam2_node.git
```
## VideoCore 라이브러리 설치
라즈베리파이를 위한 VideoCore 라이브러리의 리포지토리를 시스템에 등록
```bash
sudo add-apt-repository ppa:ubuntu-pi-flavour-makers/ppa
```
등록된 Repository 를 시스템에 반영
```bash
sudo apt update
```
라즈베리파이를 위한 VideoCore 라이브러리 설치
```bash
sudo apt install libraspberrypi-bin libraspberrypi-dev
```
## 빌드
빌드를 위해 경로를 `~/colcon_ws` 로 변경한다.
```bash
cd ~/colcon_ws
```
다음 명령으로 raspicam2 패키지를 빌드한다.
```bash
colcon build --symlink-install
```
`~colcon_build/install/local_setup.bash` 파일을 `source` 한다.
```bash
source ./install/local_setup.bash
```
raspi-config 를 이용한 카메라 인터페이스 활성화
raspi-config 설치
다음 명령으로 **raspi-config** 의 설치파일을 `/tmp` 로 가져온다.
```bash
wget https://archive.raspberrypi.org/debian/pool/main/r/raspi-config/raspi-config_20210604_all.deb -P /tmp
```
**raspi-config** 설치에 필요한 의존성들을 설치한다.
```bash
sudo apt-get install libnewt0.52 whiptail parted triggerhappy lua5.1 alsa-utils -y
```
```bash
sudo apt-get install -fy
```
## raspi-config 설치
```bash
sudo dpkg -i /tmp/raspi-config_20210604_all.deb
```
**raspi-config** 에서 카메라 인터페이스를 활성화 시키면 `boot` 파티션에서 `config.txt` 를 찾아 편집한다. 라즈비안은 부팅시 자동으로  `/dev/mmcblk0p1` 파티션을 `/boot` 폴더에 마운트한다. 그렇지만, 우분투에서는 부팅 시에 이 과정이 실행되지 않으므로 아래 명령으로 직접 마운트 시킨다. 이 부분을 하지 않으면 **raspi-config** 에서 설정을 변경하면 에러가 발생한다. 
```bash
sudo mount /dev/mmcblk0p1 /boot
```
## 라즈베리파이 카메라 인터페이스 활성화
이제 **raspi-config** 를 실행하고, 
```bash
-sudo raspi-config
```
**3 Interface Options** 메뉴를 선택 후, 그 하위 메뉴의 **P1 Camera** 메뉴를 선택하여 라즈베리파이 카메라 인터페이스를 활성화 시킨다. 
## raspicam2 노드 실행
다음 명령으로 raspicam2 노드를 실행한다.
```bash
ros2 run raspicam2 raspicam2_node --ros-args --params-file `ros2 pkg prefix raspicam2`/share/raspicam2/cfg/params.yaml
```
raspicam2 노드 토픽을 확인해보자
```bash
$ ros2 topic list
```
## [raspicam2.launch.py](http://raspicam2.launch.py) 작성
raspicam2 노드 실행을 좀더 간단히 실행할 수 있는 `raspicam2.launch.py` 파일을 작성해보자.
다음 명령으로 raspicam2 패키지 폴더로 변경한다.
```bash
cd ~/colcon_ws/src/raspicam2_node
```
`launch` 폴더 생성 후 생성된 폴더로 경로를 이동한다.
```bash
mkdir launch && cd launch
```
`[raspicam2.launch.py](http://raspicam2.launch.py)` 파일 생성
```bash
touch raspicam2.launch.py
```
적당한 편집기를 이용하여 `raspicam2.launch.py` 파일을 다음과 같이 편집, 저장한다. 
```python
import os
from ament_index_python.packages import get_package_share_directory
from launch import LaunchDescription
from launch.actions import DeclareLaunchArgument
from launch.actions import IncludeLaunchDescription
from launch.launch_description_sources import PythonLaunchDescriptionSource
from launch.substitutions import LaunchConfiguration
from launch.substitutions import ThisLaunchFileDir
from launch_ros.actions import Node
'''
package name : raspicam2
node name    : raspicam2_node
path of param: /share/raspicam2/cfg/params.yaml
'''
def generate_launch_description():
    raspicam2_param_dir = LaunchConfiguration(
        'raspicam2_param_dir',
        default=os.path.join(
            get_package_share_directory('raspicam2'),
            'cfg',
            'params.yaml'))
    return LaunchDescription([
        DeclareLaunchArgument(
            'raspicam2_param_dir',
            default_value=raspicam2_param_dir,
            description='Full path to raspicam parameter file to load'),
        Node(
            package='raspicam2',
            executable='raspicam2_node',
            parameters=[raspicam2_param_dir],
            output='screen'),
    ])
```
`raspicam2_node` 는 C++ 로 작성되었으므로 작성한 `launch` 파일을 빌드 시 포함시키려면 `CMakeList.txt` 파일에 해당 내용을 기재해 주어야 한다.
```bash
nano ~/colcon_ws/src/raspicam2_node/CMakeList.txt
```
아래 #### 으로 표시한 부분을 추가하고 저장한다. 
```bash
add_executable(raspicam2_node src/raspicam2_node.cpp)
target_link_libraries(raspicam2_node RasPiCamPublisherNode)
install(TARGETS
    RasPiCamPublisherNode raspicam
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)install(TARGETS
    raspicam2_node
    DESTINATION lib/${PROJECT_NAME}
)install(DIRECTORY cfg/ DESTINATION share/${PROJECT_NAME}/cfg)
#############################################################
install(DIRECTORY
    launch
    DESTINATION share/${PROJECT_NAME}/
)#############################################################
ament_package()
```
작성한 `launch` 파일을 반영하기 위해 다시 빌드한다. 일단 경로를 `~/colcon_ws` 로 이동한다.
```bash
cd ~/colcon_ws
```
다음 명령을 실행하여 빌드한다.
```bash
colcon build --symlink-install && source ./install/local_setup.bash
```
새로 추가한 `raspicam2.launch.py` 를 실행하여 raspicam2 노드를 구동한다.
```bash
ros2 launch raspicam2 raspicam2.launch.py
```
