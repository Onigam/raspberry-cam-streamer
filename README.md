# Raspberry Pi Camera Streaming Website

This project sets up a video streaming website on a Raspberry Pi using Raspberry Pi OS. It streams live video from a connected camera module and serves it through a responsive web interface accessible over a Tailscale VPN.

## Prerequisites

- Raspberry Pi (Model [Zero 2 W](https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/) or 3B, or newer recommended, I personnally use a [3B+](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/)). **cost: 15-40 euros**
- Micro SD card. 32 go. **cost: 5-10 euros**
- Raspberry OS installed. Check it [here](https://www.raspberrypi.com/software/) I personnally use WiFi it's more convenient to place you camera streamer so you have to set up the WiFi network and password when flashing the OS in the SDCard. You need also to activate SSH connection.
- Camera module compatible with your Raspberry Pi (e.g., Pi Camera Module with OV5647 sensor, I use [this one](https://www.amazon.fr/Jun_Electronic-Module-cam%C3%A9ra-vid%C3%A9o-Raspberry/dp/B07MNR3VM8/ref=sr_1_3?__mk_fr_FR=%C3%85M%C3%85%C5%BD%C3%95%C3%91&crid=2QP16DFA23KBI&keywords=LABISTS+B01+Raspberry+Pi+Camera+Module+5M+1080P&qid=1704711471&sprefix=labists+b01+raspberry+pi+camera+module+5m+1080p%2Caps%2C99&sr=8-3) with my 3B+). **cost: 10 euros**
- Basic knowledge of the Linux command line
- Internet connection for the Raspberry Pi
- Access to the Raspberry Pi, either directly or via SSH
- A good case for your raspberry (like [this](https://www.amazon.de/-/en/dp/B07T5L5FFN?psc=1&ref=ppx_yo2ov_dt_b_product_details)), or could be DIY (for example with [lego](https://makezine.com/article/technology/raspberry-pi/lego-raspberry-pi-enclosure/)): **cost: 0-10 euros**

## Installation and Setup

Once you have switch of your Raspberry, open a SSH connection to your Raspberry, you can use a tool like [Angry IP Scanner](https://angryip.org/download/#mac) to detect the IP of your Raspberry in your local network.

Once connected here is what you have to do

### Update Raspberry Pi OS

Ensure your system is up-to-date:

```bash
sudo apt update
sudo apt upgrade
```

### Install Necessary Packages

Install VLC and Apache2:

```bash
sudo apt install vlc apache2 -y
```

Once installed start apache2

```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```

### Set Up Camera Module

Enable the camera to your Raspberry Pi (I assume you already plugged it before booting the Raspberry):

```bash
sudo raspi-config
```

Navigate to `Interfacing Options` > `Camera` and enable the camera.

### Streaming script Setup

Create a shell script (`stream.sh`) to start video streaming:

```bash
vi stream.sh
```

Add the following content to convert the stream to `.m3u8` format, we will put it in the same web server directory than the `index.html` we will create later, here we convert the stream in HLS to be readible by the video tag of an html page:

```bash
#!/bin/bash
raspivid -o - -t 0 -w 1920 -h 1080 -fps 24 | cvlc -vvv stream:///dev/stdin --sout '#standard{access=livehttp{seglen=5,delsegs=true,numsegs=10,index=/var/www/html/stream.m3u8,index-url=http://[Your-Device-IP]/stream-########.ts},mux=ts{use-key-frames},dst=/var/www/html/stream-########.ts}' --ttl 12 --sout-keep
```

Make the script executable:

```bash
chmod +x stream.sh
```

Replace `[Your-Device-IP]` with your Raspberry Pi's IP address.

Ensure your have the correct permissions to write in `/var/www/html/`
You can change the ownership of the `/var/www/html/` directory to the user running the VLC command. Replace pi in the command below with the username if it's different:

```bash
sudo chown -R pi:pi /var/www/html/
```

### Configure Apache for HLS Streaming

Edit the Apache configuration to enable CORS. Add this to the `.htaccess` file in `/var/www/html/`:

```bash
sudo nano /var/www/html/.htaccess
```

Add the following:

```apache
<IfModule mod_headers.c>
    Header set Access-Control-Allow-Origin "*"
    <FilesMatch "\.(m3u8|ts)$">
        Header set Access-Control-Allow-Origin "*"
    </FilesMatch>
</IfModule>
```

Restart Apache to apply the changes:

```bash
sudo systemctl restart apache2
```

### Configure the streaming service

1. **Create a New Service File:**
   Create a new service file under `/etc/systemd/system/`. For example, `stream.service`:

   ```bash
   sudo nano /etc/systemd/system/stream.service
   ```

2. **Add Service Configuration:**
   In the service file, add the following configuration (replace `/home/pi/stream.sh` with the actual path to your script):

   ```ini
   [Unit]
   Description=Camera Stream Service
   After=network.target

   [Service]
   ExecStart=/home/<your_user>/stream.sh
   Restart=always
   User=<your_user>

   [Install]
   WantedBy=multi-user.target
   ```

   - `Description` is a meaningful description of the service.
   - `After=network.target` ensures the network is available when the service starts.
   - `ExecStart` is the path to your script.
   - `Restart=always` will restart the service if it fails.
   - `User` specifies the user under which the service will run.

3. **Enable and Start the Service:**
   Enable the service to start on boot and then start the service immediately:

   ```bash
   sudo systemctl enable stream.service
   sudo systemctl start stream.service
   ```

4. **Check the Service Status:**
   To check if the service is running correctly:

   ```bash
   sudo systemctl status stream.service
   ```

Using a `systemd` service gives you more control and is the recommended way to manage startup tasks on modern Linux systems like Raspberry Pi OS.

### Install your VPN client (here Tailscale)

We want to have it accessible from your personnal VPN so you can check the webpage from any of your devices that can connect to the VPN, personnally I use [Tailscale](https://tailscale.com/kb/1017/install?slug=kb&slug=1017&slug=install) which is really powerfull.

Install Tailscale to access the streaming website securely over a VPN:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the on-screen instructions to authenticate with your Tailscale account.

Once done you can check the status

```bash
tailscale status
```

Ensure it will start each time at boot

```bash
sudo systemctl enable --now tailscaled
```

#### Additional Notes:

##### Network Security:

Tailscale creates a secure network, but ensure you understand the security implications of exposing your Raspberry Pi's camera stream.

##### Performance:

Streaming over a VPN can affect performance. Monitor the stream's quality and adjust your Raspberry Pi's settings if necessary.

##### Tailscale Account:

You'll need a Tailscale account. If you don't have one, you can sign up at Tailscale's website.

### 7. Create the Webpage

Create `index.html` in `/var/www/html/`:

```bash
vi /var/www/html/index.html
```

Copy/Paste the HTML content from `./index.html` or git clone this project in your home directory and move the index.html file in `/var/www/html`

### Access the Stream

Open a web browser and navigate to the Tailscale IP of your Raspberry Pi, e.g., `http://<your-tailscale-ip>/index.html`, to view the stream.

## Troubleshooting

- Ensure the camera module is properly connected and recognized by the Raspberry Pi.
- Check the network connection and settings if the stream is not accessible.
- For issues with autoplay in browsers, verify that the video is muted.
