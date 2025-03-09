---
title: "Uncovering an Unlimited Storage Vulnerability: My First Bug Bounty"
description: Join me in my journey of discovering an unlimited storage vulnerability in Seedr.cc
author: hemantapkh
date: 2025-03-08 05:00:00 +0000
categories: [Cybersecurity]
tags: [security, bug bounty]
pin: false
math: false
mermaid: false
image:
  path: https://assets.hemantapkh.com/blog/seedr-storage-bug/thumbnail.png
---

**July 2022** While working on my open-source project, [seedrcc](https://github.com/hemantapkh/seedrcc), a Python package for interacting with [Seedr’s](https://www.seedr.cc) API, I stumbled upon a critical bug that earned me my first bug bounty.

## Uncovering the Bug

While testing the seedrcc package, I was examining different functionalities of the site using the Python package and one of them was the rename function. I attempted to rename the root folder itself, expecting an error or denial. Interestingly, the system allowed me to rename the root folder of my account.

![Seedr network tab](https://assets.hemantapkh.com/blog/seedr-storage-bug/network-tab.png)

```python
>>> ac.renameFolder('222549572', 'hemanta')
{'result': True, 'code': 200}
```

After renaming the root folder, I noticed something strange. When I checked the account's content again, a new folder with ID "222549919" had been automatically created. This folder contained a new item, "Charlie Chaplin Cruel Cruel Love (1914)" - a file that Seedr automatically downloads for new accounts. What was more unusual was the parent ID of this new folder was set to "-1", while it should have been the root folder's ID "222549572".

![Content after rename](https://assets.hemantapkh.com/blog/seedr-storage-bug/content-after-rename.png)

It seemed that the server may have misinterpreted my root folder is missing and attempted to create a new one. However, when trying to set this new folder as the root, something might have gone wrong, as my original root folder still existed. 

Unlike regular folders, which reside within the account's root folder and contribute to the account's storage, this new folder existed independently. This meant I had access to a folder outside my root structure, where files didn't count against my storage limit. This allowed me to download as many items as I wanted within this new folder, bypassing the storage restriction.

```python
>>> ac.addTorrent(<magnet_link>, folderId='222549919')
```

When browsing content from the Seedr site, no files would show because it only displayed the content of my root folder, which was empty. However, by using the API or going to [https://www.seedr.cc/files/222549919](https://www.youtube.com/watch?v=hvL1339luv0){:target="_blank"}, I could access all the contents of the new folder.

```python
>>> ac.listContents('222549919')
```

![Content of the new folder](https://assets.hemantapkh.com/blog/seedr-storage-bug/new-folder-content.jpg)

## Reporting the Issue

**July 5** Recognizing the potential impact of this bug, I promptly reported it to the Seedr team and got this response on the very next day. 🎉

![Seedr Reply Email](https://assets.hemantapkh.com/blog/seedr-storage-bug/seedr-email-light.png){: .light }
![Seedr Reply Email](https://assets.hemantapkh.com/blog/seedr-storage-bug/seedr-email-dark.png){: .dark }
