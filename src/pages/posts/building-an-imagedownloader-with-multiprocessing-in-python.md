---
title: Building an ImageDownloader with multiprocessing in python
subtitle: Sometimes it takes hours to download a large number of images — let’s fix that
date: 2019-12-04T08:25:00.010Z
thumb_img_path: /images/1_f4akt8pdljhllham3ln4dw.jpeg
content_img_path: /images/1_f4akt8pdljhllham3ln4dw.jpeg
canonical_url: ''
template: post
---
Want to skip the post and see the full code directly: here is the github repo.

[GitHub ](https://github.com/amitupreti/image_downloader_multiprocessing_python)

I get it, you are tired of waiting for your program to download images. It takes hours sometimes when I have to download thousands of images and you possibly can't keep waiting for your program to finish downloading these dumb images. You have a lot of important stuff to do. 

Let's build a simple image downloader script that will read a text file and download all the images listed nicely in a folder superfast.

## Final result

This is what we will be building at the end.

![cat images downloading](https://snipboard.io/VOXItq.jpg)

![cat image downloading](https://snipboard.io/6UgtE2.jpg)

#### Install the dependencies

let's install the everybody's favorite _requests_ library.

```bash
pip install requests 
```

Now, we will see some basic code to download a single URL and try to automatically find the name of the image and also how to use retries.

```python
import requests

res = requests.get(img_url, stream=True)
count = 1
while res.status_code != 200 and count <= 5:
    res = requests.get(img_url, stream=True)
    print(f'Retry: {count} {img_url}')
    count += 1
```

Here we retry to download the images 5 times in case it fails. Now let's try to automatically find the name of image and save it

```python
#import more required library

import io
from PIL import Image

#lets try to find the image name
image_name = str(img_url[(img_url.rfind('/')) + 1:])
if '?' in image_name:
    image_name = image_name[:image_name.find('?')]
```

#### Explanation

Let's consider the url we are trying to download is 

_https://instagram.fktm7-1.fna.fbcdn.net/vp/581ba4467732636cab13a38c7bd9f796/5E4C7217/t51.2885-15/sh0.08/e35/s640x640/65872070_1200425330158967_6201268309743367902_n.jpg?_nc_ht=instagram.fktm7-1.fna.fbcdn.net&_nc_cat=111_

Well, this is a mess. Let's break what code does to the url. We first find the last frontslash(`/`) with `rfind` and then select everything after that. This is the result

_65872070_1200425330158967_6201268309743367902_n.jpg?_nc_ht=instagram.fktm7-1.fna.fbcdn.net&_nc_cat=111_

Now our second part finds a `?` and then only takes whatever is in front of it.

This is our final image name

_65872070_1200425330158967_6201268309743367902_n.jpg_

This result is pretty good and will work for most of cases.

Now that we have our image name and image downloaded. We will now save it.

```python
i = Image.open(io.BytesIO(res.content))
i.save(image_name)
```

If you are thinking how the heck am I supposed to use the code above, then you are thinking correct. Here is a pretty function with everything we did above squished.  Here we also tested if the downloaded type is an image and in case we couldn't find the image name

```python
def image_downloader(img_url: str):
    """
    Input:
    param: img_url  str (Image url)
    Tries to download the image url and use name provided in headers. Else it randomly picks a name
    """
    print(f'Downloading: {img_url}')
    res = requests.get(img_url, stream=True)
    count = 1
    while res.status_code != 200 and count <= 5:
        res = requests.get(img_url, stream=True)
        print(f'Retry: {count} {img_url}')
        count += 1
    # checking the type for image
    if 'image' not in res.headers.get("content-type", ''):
        print('ERROR: URL doesnot appear to be an image')
        return False
    # Trying to red image name from response headers
    try:
        image_name = str(img_url[(img_url.rfind('/')) + 1:])
        if '?' in image_name:
            image_name = image_name[:image_name.find('?')]
    except:
        image_name = str(random.randint(11111, 99999))+'.jpg'

    i = Image.open(io.BytesIO(res.content))
    download_location = 'cats'
    i.save(download_location + '/'+image_name)
    return f'Download complete: {img_url}'
```

Now, "where is this multiprocessing this guy was talking about?" you might say.

This is very simple. We will simply define our pool and pass our function and image URL to it.

```python
results = ThreadPool(process).imap_unordered(image_downloader, images_url)
for r in results:
    print(r)
```

Let's put this in a function

```python
def run_downloader(process:int, images_url:list):
    """
    Inputs:
        process: (int) number of process to run
        images_url:(list) list of images url
    """
    print(f'MESSAGE: Running {process} process')
    results = ThreadPool(process).imap_unordered(image_downloader, images_url)
    for r in results:
        print(r)
```

Again, you might say this is all okay stuff. But I want to start to download my list of 1000 images right away. I don't want to copy and paste these codes and try figuring out merging everything.

Here is a full script. It does the following

1. Takes image list text file and number of the process as input
2. Downloads them as per your desired speed.
3. And also prints the total time taken to download the files.
4. There are also a few nice functions that helps us to read the filename and handle errors and stuff.

### Full script

```python
# -*- coding: utf-8 -*-
import io
import random
import shutil
import sys
from multiprocessing.pool import ThreadPool
import pathlib

import requests
from PIL import Image
import time


start = time.time()


def get_download_location():
    try:
        url_input = sys.argv[1]
    except IndexError:
        print('ERROR: Please provide the txt file\n$python image_downloader.py cats.txt')
    name = url_input.split('.')[0]
    pathlib.Path(name).mkdir(parents=True, exist_ok=True)
    return name


def get_urls():
    """
    Returns a list of urls by reading the txt file supplied as argument in terminal
    """
    try:
        url_input = sys.argv[1]
    except IndexError:
        print('ERROR: Please provide the txt file\n Example \n\n$python image_downloader.py dogs.txt \n\n')
        sys.exit()
    with open(url_input, 'r') as f:
        images_url = f.read().splitlines()

    print('{} Images detected'.format(len(images_url)))
    return images_url


def image_downloader(img_url: str):
    """
    Input:
    param: img_url  str (Image url)
    Tries to download the image url and use name provided in headers. Else it randomly picks a name
    """
    print(f'Downloading: {img_url}')
    res = requests.get(img_url, stream=True)
    count = 1
    while res.status_code != 200 and count <= 5:
        res = requests.get(img_url, stream=True)
        print(f'Retry: {count} {img_url}')
        count += 1
    # checking the type for image
    if 'image' not in res.headers.get("content-type", ''):
        print('ERROR: URL doesnot appear to be an image')
        return False
    # Trying to red image name from response headers
    try:
        image_name = str(img_url[(img_url.rfind('/')) + 1:])
        if '?' in image_name:
            image_name = image_name[:image_name.find('?')]
    except:
        image_name = str(random.randint(11111, 99999))+'.jpg'

    i = Image.open(io.BytesIO(res.content))
    download_location = get_download_location()
    i.save(download_location + '/'+image_name)
    return f'Download complete: {img_url}'


def run_downloader(process:int, images_url:list):
    """
    Inputs:
        process: (int) number of process to run
        images_url:(list) list of images url
    """
    print(f'MESSAGE: Running {process} process')
    results = ThreadPool(process).imap_unordered(image_downloader, images_url)
    for r in results:
        print(r)


try:
    num_process = int(sys.argv[2])
except:
    num_process = 10

images_url = get_urls()
run_downloader(num_process, images_url)


end = time.time()
print('Time taken to download {}'.format(len(get_urls())))
print(end - start)
```

##### save it to a python file and let's run it.

```
python3 image_downloader.py cats.txt
```

#### Dude give me a Github link.

Here is a link to the Github repo
https://github.com/nOOBIE-nOOBIE/image_downloader_multiprocessing_python

#### Dude teach me how to use it

#### Usage

```
python3 image_downloader.py <filename_with_urls_seperated_by_newline.txt> <num_of_process>
```

This will read all the urls in the text file and download them into a folder with name same as the filename.

num_of_process is optional.(by default it uses 10 process)

### Example

```
python3 image_downloader.py cats.txt
```

![cat images downloading](https://snipboard.io/VOXItq.jpg)

![cat image downloading](https://snipboard.io/6UgtE2.jpg)

Would love any response on how this can be improved further.

#### Happy coding!

This was also posted in[ dev.to](https://dev.to/amitupreti/building-an-imagedownloader-with-multiprocessing-in-python-1fn2) and [medium](https://medium.com/better-programming/building-an-imagedownloader-with-multiprocessing-in-python-44aee36e0424).
