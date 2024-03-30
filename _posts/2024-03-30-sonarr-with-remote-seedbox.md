---
title: Setting up Sonarr with a remote seedbox
date: 2024-03-30 13:00:00 +0000
categories: [Homelab]
tags: [sonarr,media,seedbox]     # TAG names should always be lowercase
---

I recently picked up a seedbox to help build ratio on private trackers since my upload speed isn't great. I figured I'd document the process in case it comes in handy for others or myself in the future.

## Setting up qBittorrent on Sonarr
I use Ultra.cc as my seedbox provider so these instructions may differ slightly if you use a different provider.

Head over to your seedbox provider and find the information for your qBittorrent instance. In my case, it'll look something like the below:

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/ultra-qbittorrent-info.png)

Head over to Settings > Download Clients > Add Client (+) and set up your client. The information below will match what has been provided by my provider as above.

```
Name: qBittorrent - Ultra.CC
Host: test.proto.usb.io
Port: 88888
Use SSL: Unchecked (Provider dependent)
Username: chunaki7
Password: ******
Category: tv-sonnar (Optional)
```
> I'd recommend setting a category to help manage the files once you're finished with them.
{: .prompt-tip }

Click Test and if it's all OK, Save.

### Remote Path Mapping
Sonarr needs to be told where qBittorrent places completed downloads and where to look for them on your local machine. As qBittorrent and Sonarr are on different machines, you'll need to tell it where these files can be found by setting up remote path mappings. 

Remote Path Mapping acts as a dumb find Remote Path and replace with Local Path. A remote path map is a DUMB search/replace (where it finds the REMOTE value and replaces it with LOCAL value for the specified Host).[^1]

Under Settings > Download Clients, you'll see Remote Path Mappings. Press the plus (+) icon in the bottom right as below.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/remote-path-add.png)

A small window will popup as below:

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/remote-path-add-2.png){: width="500" height="400" }


* `Host`: Select the download client that you've just setup as per the first section.

* `Remote Path`: The folder which qBittorrent stores the completed files. (We'll set this up later on the seedbox)

* `Local Path`: The folder which Sonarr will look at for any media files.


## Tweaks to qBittorrent
If you've followed as above, I've set a category of `tv-sonarr` on the download client under Sonarr. We'll just need to change a few things on qBittorrent so that it matches this setup.

### Creating the download folder
We'll need to create the folder for downloads to be stored. We'll need to SSH into the machine using the details given to us by the seedbox provider.
```bash
ssh chunaki7@test.proto.usb.io
```
Once logged on, create the folder by running the comnmand
```bash
mkdir /home/test/downloads/qbittorrent/sonarr-tv/
```

### Setting up categories
On qBittorrent, you'll find a section called `CATEGORIES` in the left panel. Right click and select Add Category.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/category-1.png)

A new window will open up. We'll setup the category that we defined under the download client and set the download location.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/category-2.png)

### Keeping incomplete files in a temp folder
Ultra.cc suggest that we don't point Syncthing to an active download directory. In this case, I like to set the option to keep incomplete downloads in a folder called `temp`. As we have automatic torrent handling setup, when the download is completed, it'll automatically move over to the folder that we've set for the category. 

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/qbittorrent-options.png)

## Setting up Syncthing
I won't be running through how to install Syncthing as it assumes that it's already been installed on your seedbox and your local machine.

### Adding Folders
On opening the Syncthing instance, you'll be prompted with window about `Allow Anonymous Usage Reporting?`. Select Yes/No depending on your preference.[^2]

#### Seedbox Syncthing
Set up the folder which you want to sync. In our case, this will be the folder `/home/test/downloads/qbittorrent/sonarr-tv/`. 

Head over to Folders and select Add Folder.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-1.png)

Under General, we'll set up the below:
```
Folder Label: sonarr-tv
Folder Path: /home/test/downloads/qbittorrent/sonarr-tv/
```


Under Advanced, we'll set up the below:
```
Scanning > Watch for Changes: Unchecked
Full Rescan Interval (s): 3600
Folder Type: Send Only
```

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-2.png)

#### Local machine Syncthing

Similar as above, we'll net to set up a folder that syncs with the seedbox. 

Under General, we'll set up the below:
```
Folder Label: sonarr-tv
Folder Path: /data/torrents/tv/
```
> The `folder path` is where we'll be storing our completed downloads on the local machine
{: .prompt-info }

Under Advanced, we'll set up the below:
```
Scanning > Watch for Changes: Unchecked
Full Rescan Interval (s): 3600
Folder Type: Receive Only
File Pull Order: Random
```

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-6.png)


### Setting up syncing between both machines

On your local machine Syncthing instance, get the Device ID.
Go to Actions > Show ID.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-3.png)

You'll be presented with the below. Click Copy.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-4.png)

On your seedbox Syncthing instance, go to Remote Devices > Add Remote Device. 

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-5.png)

```
GENERAL
Device ID: The ID you copied above
Device Name: Any name for this device

SHARING
Check the folder that you added earlier

ADVANCED
Device rate limits (Ultra.cc ask you to set this as per their fair use policy)
Incoming Rate Limit (KiB/s): 20000
Incoming Rate Limit (KiB/s): 20000
```
![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-7.png)


Set up your local machine Syncthing instance the same way but copy the seedbox Syncthing device ID instead.
You should then see the below and that they're both ready to sync.

![img](/assets/img/posts/2024-03-30-sonarr-with-remote-seedbox/syncthing-8.png)

Downloads on the seedbox will now be synced to your local machine where Sonarr can import.


## References

[^1]: [Servarr Wiki - Remote Path Mappings](https://wiki.servarr.com/sonarr/settings#remote-path-mappings)
[^2]: [Ultra.cc Syncthing Documentation](https://docs.ultra.cc/books/syncthing/page/syncthing)