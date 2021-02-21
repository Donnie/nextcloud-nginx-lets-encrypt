I came to know about Nextcloud about a year or so back, but then I could not believe it would come to grow to such an extent as we see today.

Nextcloud comes with every kind of tool that you may expect from a Google Cloud or Onedrive. Emails, Calendars, Chatting, Uncompressed Photo storage and sharing, so on and so forth. And like Wordpress new apps can be added thereby extending the suite of functionalities it has to offer.

And of course Nextcloud is GDPR and HIPAA compliant, and if you are self-hosting it, there is absolute privacy.

So I finally decided to give it a go.

I had some requirements though from the whole setup.

1. I wanted it to be set up on my domain: cloud.donnie.in
2. It should have full A+ SSL
3. Should not cost me more than 3-4 EUR a month
4. I should be able to use the server for my other small projects.
5. I should be able to activate the Face Recognition applet which needs around 2GB of RAM
6. Should be able to add Backblaze B2 object storage
7. Send and Receive email

And while I was at it, installing Nextcloud manually on a personal VPS server, I noted everything down, so as to be able to share the experience/knowledge. You do not need an engineering degree, or have heard the word PHP before. You can blindly follow the steps, I would of course explain each one of them. But if you are a pro, I wish you can improve this guide in some way. Point out mistakes or redundancies if found or suggest improvements too.

## Getting a personal Cloud.
Depending on where you are right now, I would suggest getting a server that's located very close to you. That keeps the latencies low.

Of course if you do not mind the few ms of latency, the world is your playground.

### For India
Digital Ocean Bangalore could be great, a server with 1GB RAM costs $5/month.

### For USA
There are several locations from vultr.com, and I believe no one can beat vultr at pricing. You can get a server with 512MB at $3.5/month

There is something more to US, you can get a forever free server with 512MB RAM from Google Cloud, and from Amazon you can get a similar offer for one year free of cost.

### For Germany
And this is a top secret, not many people know about Hetzner. Hetzner has locations in Nurenberg, Falkenstein and Helsinki. Which are very close to Berlin, for me. Also the pricing ist unschlagbar: 2.96 EUR/month for an Ubuntu server with 2GB RAM!

## First things first

### Update
The first things to do on a new server is always update the system:

```
sudo apt-get update

sudo apt-get upgrade
```

### Install Swapfile
Nextcloud says they need only about 128 MB of RAM, but that's like saying a snake needs to eat only twice a year. Of course more RAM makes it possible to do multi tasking, faster processing etc. As my plan includes the Face Recognition applet I need to eke out as much as possible from the system.

Given that I have 20GB SSD space I can afford to give away 3GB for swap.

#### Allocating a 3GB /swapfile on root
```
sudo fallocate -l 3G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Now if you do `free -h` you would see there is a 3GB swap added to available memory.

#### We also need to make sure the swapfile is loaded at every reboot.

Create a backup of fstab
```
sudo cp /etc/fstab /etc/fstab.bak
```
Add the swapfile to fstab
```
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Reboot `reboot` and check memory `free -h` to confirm

### Update timezone
Most VPS come with UTC as default timezone, this might create some errors if you plan to use your email service with Nextcloud and you are not on the UTC timezone.

Update time zone like so
```
timedatectl set-timezone "Europe/Berlin"
```

If you are not sure about your timezone you can do `timedatectl list-timezones` to get a list of them. However, for India it is `"Asia/Kolkata"`

