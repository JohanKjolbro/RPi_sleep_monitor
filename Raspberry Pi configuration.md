---
annotation-target:
---
# Step 1 : Raspberry Pi OS

Download https://www.raspberrypi.com/news/raspberry-pi-imager-imaging-utility/ 
Use it to flash raspberry pi os onto an SD card

# Step 2 : Enable automatic hotspot


> You will need a monitor, mouse and keyboard for the initial setup

## Configure hotspot

Now that your Raspberry Pi is up and running, it’s time to transform it into a hotspot.

### Select your WLAN country

In terminal open raspi-config :

~~~
sudo raspi-config
~~~

Select **localisation options** and **WLAN Country**

### Find your USB Wi-Fi adapter

First, we need to find the USB adapter. Run the following command to identify Wi-Fi devices with the Network Manager CLI:

```shell-session
$ nmcli device
```

You should see output similar to the following:

```text
DEVICE         TYPE      STATE                   CONNECTION
wlan1          wifi      connected               Example Wi-Fi
lo             loopback  connected (externally)  lo
wlan0          wifi      disconnected            --
p2p-dev-wlan0  wifi-p2p  disconnected            --
eth0           ethernet  unavailable             --
```

In the above output, the USB Wi-Fi module, `wlan1`, is connected to a Wi-Fi network named "Example Wi-Fi". The built-in Wi-Fi device, `wlan0`, is not currently in use, so the state currently reads "disconnected".

If your Raspberry Pi has a built-in Wi-Fi module, it should show up by default as `wlan0`. The first Wi-Fi module you connect should show up as `wlan1`, and subsequent adapters will display as `wlan2`, `wlan3`, and so on. Depending on your specific configuration, your Raspberry Pi might connect to your network using either the USB adapter or the built-in Wi-Fi module.

### Create hotspot network

Next, we’ll use the built-in Wi-Fi module to broadcast a hotspot network. Run the following command to create a hotspot, replacing the `<hotspot name>` and `<hotspot password>` placeholders with a hotspot name and password of your choice:

```shell-session
$ sudo nmcli device wifi hotspot ssid <hotspot name> password <hotspot password> ifname wlan0
```


After creating the hotspot network, your hotspot should automatically become active.

Next, connect to the hotspot Wi-Fi network from your usual computer. Look for a network with an SSID matching the hotspot name you chose in the previous step. Use the password you also provided in that step to authenticate.

Then, connect to your Raspberry Pi using SSH:

```shell-session
$ ssh <username>@pi-hotspot.local
```

And run the following command to view your current connections:

```shell-session
$ nmcli connection
```

You should see output similar to the following:

```text
NAME       UUID                                  TYPE      DEVICE
Hotspot    69d77a03-1cd1-4ec7-bd78-2eb6cd5f1386  wifi      wlan0
lo         f0209dd9-8416-40a0-971d-860d3ff3501b  loopback  lo
Ethernet   4c8098c7-9f7d-4e3e-a27a-70d54235ec9a  ethernet  --
Example 1  f0c4fbcc-ac88-4791-98c2-e75685c70e9f  wifi      --
Example 2  9c6098a7-ac88-40a0-5ac2-b75695c70e9e  wifi      --
```

The connection named `Hotspot` represents your new hotspot network. The `Example 1` and `Example 2` connections above represent saved Wi-Fi connections which are inactive.

### Configure hotspot network

Let’s configure your hotspot network to automatically broadcast whenever your Raspberry Pi boots. When your Raspberry Pi boots, it starts whichever connection has autoconnect enabled with the highest priority. To ensure that your hotspot always starts on boot, we’ll enable autoconnect for the hotspot and configure a priority higher than any other connection.

Re-run the `nmcli connection` command above and copy the UUID for your hotspot network from the table. Then, run the following command to view connection properties for your hotspot network, replacing the `<hotspot UUID>` placeholder with the UUID for your hotspot:

```shell-session
$ nmcli connection show <hotspot UUID>
```

The output will contain a lot of properties that describe your hotspot network. But we’re only interested in the following two properties for now:

```text
connection.autoconnect:                 no
connection.autoconnect-priority:        0
```

Run the following command to change the priority and autoconnect properties for your hotspot, replacing the `<hotspot UUID>` placeholder with the UUID for your hotspot that you copied to your clipboard before:

```shell-session
$ sudo nmcli connection modify <hotspot UUID> connection.autoconnect yes connection.autoconnect-priority 100
```

If your command executed successfully, we should see the following new values for those properties when we re-run `nmcli connection show <hotspot UUID>`:

```text
connection.autoconnect:                 yes
connection.autoconnect-priority:        100
```

# Step 3 : Enable VNC

~~~
sudo raspi-config
~~~

Go to **Interface options** and go to **VNC** and enable it

# Install libfreenect2-Raspberry-Pi4-support and freenect2-python

On the RPi, go to home :
~~~
cd ~
~~~

Clone the repo :
~~~
git clone https://github.com/Jyurineko/libfreenect2-Raspberry-Pi4-support
~~~

Now follow the steps on the repo's Readme :
~~~
sudo apt install libudev-dev  
sudo apt install mesa-utils  
sudo apt update  
sudo apt install libgles2-mesa-dev

sudo apt-get install build-essential cmake pkg-config
sudo apt-get install libusb-1.0-0-dev  
sudo apt-get install libturbojpeg0-dev  
sudo apt-get install libglfw3-dev  
sudo apt-get install libopenni2-dev 

cd ~/libfreenect2-Raspberry-Pi4-support
 
mkdir build && cd build  
cmake .. -DCMAKE_BUILD_TYPE=Release -DENABLE_CUDA=OFF -DENABLE_OPENCL=OFF -DENABLE_OPENGL_AS_ES31=ON -DENABLE_CXX11=ON -DENABLE_VAAPI=OFF

make -j4  
sudo make install  
sudo ldconfig  
sudo cp ../platform/linux/udev/90-kinect2.rules /etc/udev/rules.d/
~~~

After install, you should have a new folder called freenect2 in your home directory. You need to copy it's contents into the right places :

~~~
cd ~/freenect2

sudo cp ./include /usr/local/include
sudo cp ./lib /usr/local/lib
~~~

Set the environment variables used during install and execution in your .bashrc :

~~~
sudo vim ~/.bashrc
~~~

Add the following two lines :

~~~vim

export PKG_CONFIG_PATH=/home/projet/libfreenect2-Raspberry-Pi4-support/
export LD_LIBRARY_PATH=/usr/local/lib/

~~~

Exit vim, then source the .bashrc :

~~~terminal
source ~/.bashrc
~~~

Now, we need to install the python package :
~~~
pip install freenect2 --break-system-packages
~~~

# Step 4 : Test

Check to see if Protonect works :
~~~
~/libfreenect2-Raspberry-Pi4-support/build/bin/Protonect
~~~

Check to see if the python interface works. Create a file and give it permissions :

~~~
cd ~
touch test.py
sudo chmod +rwx test.py
sudo vim test.py
~~~

Paste the following code :

~~~vim
# Import parts of freenect2 we're going to use
from freenect2 import Device, FrameType

# We use numpy to process the raw IR frame
import numpy as np

# We use the Pillow library for saving the captured image
from PIL import Image

# Open default device
device = Device()

# Start the device
with device.running():
    # For each received frame...
    for type_, frame in device:
        # ...stop only when we get an IR frame
        if type_ is FrameType.Ir:
            break

# Outside of the 'with' block, the device has been stopped again

# The received IR frame is in the range 0 -> 65535. Normalise the
# range to 0 -> 1 and take square root as a simple form of gamma
# correction.
ir_image = frame.to_array()
ir_image /= ir_image.max()
ir_image = np.sqrt(ir_image)

# Use Pillow to save the IR image.
Image.fromarray(256 * ir_image).convert('L').save('output.jpg')
~~~

Exit vim and run the code :

~~~
python3 ./test.py
~~~

