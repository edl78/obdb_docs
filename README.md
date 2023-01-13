# High level description of the OBDB project repos and how to use them
You can skip directly to Workflow below to get an idea of what can be done with the OBDB repos. For some more details continue reading here.

## Overview of repos
There are 4 github repos that belong to the project (and this repo just for some documentation).

### dev_tips
Short internal description of how we used to setup Docker container remote development with VScode during the pandemic. Very handy in times of working from home to office servers. Repo with description [here](https://github.com/edl78/dev_tips).

### weed_training
The [weed_training](https://github.com/edl78/weed_training) repo contians PyTorch code for training a network to do weed detection. Includes Bayesian hyperparameter optimization.

### weed_annotations
The [weed_annotations](https://github.com/edl78/weed_annotations) repo contains code which sets up a new pipeline for doing iterative semi-automatic annotations. It was developed to communicate with [CVAT](https://github.com/opencv/cvat) through the API. This work was done during the API 1.0 time period of CVAT, i.e. CVAT version up to 1.7.0, and is the reason why you will need to use an old version of CVAT (see further instructions).  Of course you could always update the code to support CVAT API 2.0 and if you do please consider sharing your updates back to us!

Has interface to cvat for fetching annotations and inserting them into a MongoDB external from CVAT. Also a mongoExpress web interface to MongoDB for easy access to this database.

### analytics
The [analytics](https://github.com/edl78/analytics) repo contains a few helper functions we used during the project to study our data:
- T-sne clustering of bboxes for each task. GPU accelerated via Rapids lib installed via pip and accessed via an http-api made with flask.

Please note that older versions of the code uses conda and for these version the following notes apply:
- The Rapids library is only available in conda if we do not like to build from source. This gives double virtualization since we also use Docker for every function/repo.
- The Analytics conda part is notoriously hard to build due to packages being deprecated very fast. Expect to debug and work hard to get it to build. Build failure logs are also very hard to decrypt.

## Workflow

There are two workflows described here, the *fast track to training* and the *complete setup* which includes analytics, annotation framework (CVAT + our own code to interface with CVAT) etc. Common steps for the two workflows are getting code and data, building containers and setting up any services.

### Getting the code
- git clone repos listed above.

### Getting the data
Use the docker-compose files in the cloned code to get images, annotations, auxillary data (pathnames etc), calibration data and pre-trained models, see `readme.md` in [weed_training](https://github.com/edl78/weed_training)
- First decide where to store the images (will use 2 times 670GB for 4K images, 2 times 115GB for full-HD images), edit the `.env`-file and set the variable `WEED_DATA_PATH` to where **on the host machine** you decided to store the images. This will be mapped to `/weed_data` inside the container. Images will be downloaded and exracted into a sub-directory called `fielddata`.
- Annotations are stored in Pickle-files. To download the annotations 
- Please put 
- Calibration data **TBD**
- Artefacts folder contains a pretrained resnet18 Pytorch model and json files to import to MongoDB for the short path to training. Json files are meta.json, tasks.json and annotation_data.json. Pickle files for fast track to training are also supplied.



### Building all Docker images
- Build the Docker images by following the respective repos build instuctions. This applies to [weed_training](https://github.com/edl78/weed_training) and [weed_annotations](https://github.com/edl78/weed_annotations),  however [analytics](https://github.com/edl78/analytics) is optional, and obdb_docs (this repository) as well as [dev_tips](https://github.com/edl78/dev_tips) are just for documentation.

### Start services
- How to start all services needed is covered in each repo. If fast track is chosen only weed_training is needed. If complete setup is prefered start Analytics (optional), weed_annotations and last weed_training. Analytics is not bundled with weed_annotations as it needs a GPU with at least 16GB of memory for the T-sne analysis. Weed_annotations need quite a lot of CPU RAM to hold all dashboard data from the statistics and analytics, therefore they do not share docker-compose. If running on a more capable server feel free to put them in the same docker-compose. Then weed_annotations depends on analytics.

### Clone and start CVAT

This is, as mentiond above, only needed if you like to inspect, modify or add annotations to the dataset. CVAT is not needed if you will follow the fast track to training below.
- Upload tasks to cvat code found in the weed_training repo under code. `docker-compose -f docker-compose-upload-train-data-cvat.yml up` and `docker-compose -f docker-compose-upload-val-data-cvat.yml up`


## Fast-track to training vs complete setup

### Fast-track to training
- Use premade Pickle-files `pd_train_full_hd.pkl` and `pd_val_full_hd.pkl`
- No need to setup the analytics service
- start training as per instructions in the weed_training repo.
- get metrics, see instructions in the weed_training repo.

### Complete setup of training and possibility to annotate
- Build weed_annotations and run as per instructions in the repos. Build weed_training but do not run yet.
- There are two ways to load annotations into MongoDB.
- Alternative 1: Load annotations into mongo with mongo interface. Install the MongoDB Database Tools by downloading from mongodb website and follow install instructions. Find the mongodb json files in the artefacts folder downloaded from the OBDB site. To import data into MongoDB use (fill in your username, password and port):
- Known bugs in the bitnami/mongodb: must initialize with port 27017 and do not change root user name! Otherwise it will not work...
- https://www.mongodb.com/try/download/database-tools
- For the annotation data: `mongoimport --username= --password= --host=localhost --port= --collection=annotation_data --db=annotations annotation_data.json`
- For the meta data:
`mongoimport --username= --password= --host=localhost --port= --collection=meta --db=annotations meta.json`
- For the tasks data:
`mongoimport --username= --password= --host=localhost --port= --collection=tasks --db=annotations tasks.json`
- Alternative 2: - Fetch all annotations via dashboard (available after starting weed_annotations) to MongoDB: press update annotations button in the dashboard running on localhost:8050
- MongoDB contents can be viewed via localhost:8081, on the MongoExpress GUI.
- Start training and get metrics are covered in the weed_training repo.
- Auto annotation is available in the weed_training repo to speed up annotations. Find more info in the weed_training repo.


### Finding new optimal hyperparameters
- The HPO search is covered in the weed_training repo, but is controlled via the settings file.

### Just want to play?
- Use the pretrained resnet18.pth with supplied class_map so you know what class predictions mean and implement an inference solution of your choice.
- Use the same normalization as the dataloader during training, image resolution is expected to be 1920x1080 in png format.


## Notes on the data

### 4K vs full HD (1920x1080)
The GoPro Hero 6 Black cameras were used in the project to record video in 4k from the fields. Due to the installation geometry of the cameras in the tractor most of the interesting bits in the images are found in the lower middle (full-HD)part of the video. For this reason, and to speed up CVAT as well as subsequent training, only the lower half and the middle 2/4 of the image is used (i.e. 1/4 on each side of the image are dropped) in our work.

There still are quite a few annotations in the rest of the 4k part outside the full hd frame for some of the images. These annotations are automatically cropped if they cross the 4k to full hd boundary, the ones fully outside are fully dropped.

### Directory structure and file naming
The published data still keeps some of the directory structure that was used internally during the project and this might confuse you. There are two top-level directories:

`tractor-31-extracted` contians 4K frames extracted from the 4K videos stored in png-files.
`tractor-32-cropped` contians full-HD frames cropped from any similarly named 4K images found under `tractor-31-extracted` . Again stored in png-files.

For the rest of the directory structure, lets look at these two examples:

`tractor-32-cropped/20200515101008/2L/GH070066/<imagename>.png`
`tractor-31-extracted/20190604131554/3R/GH010049/<imagename>.png`

Following the top-level directory there is a date and time coded in the foldername according to YYYYMMddHHMMss (see `man date` in Linux).

Then follows directories `1L`, `1R`, `2L`, `2R`, `3L`, and  `3R`. These are a numbering for the six individual cameras used simultaneously.

Lastly in the directory structure, the name of the GoPro video file follows, e.g. `GH070066` and `GH010049`.

For each video file captured by the GoPro we used [ffmpeg](https://ffmpeg.org/) to extract key frames in sequence. The naming of the images is just a sequence of numbers for the key frames.

### Extraction of frames from video

Keyframes were extracted from the video files using the following call to ffmpeg:

`ffmpeg -skip_frame nokey -vsync 0 -i <videofile>.MP4 -r 30 -y -an -f image2 'iframes/<videofile>frame%04d.png'`

At some dates video was captured with 30 FPS and for later dates 25 FPS was used instead. This requires changing the above to use `-r 25`.

#### ffmpeg version used
Installed via apt on Ubuntu 18.04, `apt list --installed` gives:
`ffmpeg/now 7:3.4.6-0ubuntu0.18.04.1 amd64`.

Running:
`ffmpeg -version` results in this output:

`ffmpeg version 3.4.6-0ubuntu0.18.04.1 Copyright (c) 2000-2019 the FFmpeg developers
built with gcc 7 (Ubuntu 7.3.0-16ubuntu3)
configuration: --prefix=/usr --extra-version=0ubuntu0.18.04.1 --toolchain=hardened --libdir=/usr/lib/x86_64-linux-gnu --incdir=/usr/include/x86_64-linux-gnu --enable-gpl --disable-stripping --enable-avresample --enable-avisynth --enable-gnutls --enable-ladspa --enable-libass --enable-libbluray --enable-libbs2b --enable-libcaca --enable-libcdio --enable-libflite --enable-libfontconfig --enable-libfreetype --enable-libfribidi --enable-libgme --enable-libgsm --enable-libmp3lame --enable-libmysofa --enable-libopenjpeg --enable-libopenmpt --enable-libopus --enable-libpulse --enable-librubberband --enable-librsvg --enable-libshine --enable-libsnappy --enable-libsoxr --enable-libspeex --enable-libssh --enable-libtheora --enable-libtwolame --enable-libvorbis --enable-libvpx --enable-libwavpack --enable-libwebp --enable-libx265 --enable-libxml2 --enable-libxvid --enable-libzmq --enable-libzvbi --enable-omx --enable-openal --enable-opengl --enable-sdl2 --enable-libdc1394 --enable-libdrm --enable-libiec61883 --enable-chromaprint --enable-frei0r --enable-libopencv --enable-libx264 --enable-shared
libavutil      55. 78.100 / 55. 78.100
libavcodec     57.107.100 / 57.107.100
libavformat    57. 83.100 / 57. 83.100
libavdevice    57. 10.100 / 57. 10.100
libavfilter     6.107.100 /  6.107.100
libavresample   3.  7.  0 /  3.  7.  0
libswscale      4.  8.100 /  4.  8.100
libswresample   2.  9.100 /  2.  9.100
libpostproc    54.  7.100 / 54.  7.100`


## Notes on CVAT
The code in these repos was developed when CVAT was still on API 1.0, and hence an older version of CVAT needs to be used. You only need CVAT if you like to inspect, modify or add annotations to the dataset. CVAT is not needed if the fast track to training is choosen (see below).

Follow these steps to the get the last version of CVAT with API 1.0 support running on your machine (these steps work for v1.7.0):

- Clone the cvat repo: `git clone https://github.com/opencv/cvat.git`
- Use a version with API 1.0 support, e.g. change to the 1.7.0 branch: `git checkout v1.7.0`
- Pull older versions of cvat_server and cvat_ui, e.g. for CVAT 1.7.0: `docker pull openvino/cvat_server:v1.7.0` and `docker pull openvino/cvat_ui:v1.7.0`
- Modify `docker-compose.yml` and set the chosen version's tag for the two openvino images:
```json
cvat:
    container_name: cvat
    image: openvino/cvat_server:v1.7.0
```
```json
cvat_ui:
    container_name: cvat_ui
    image: openvino/cvat_ui:v1.7.0
```
- Put the images you downloaded in a folder named fielddata and map its parent folder as the `<shared_folder>` as per the changes below (to docker-compose.yml). This is imporant as it ensures that paths will be correct for published annotations as their paths originate from the fielddata folder.

```json
services:
...
  cvat:
...
    volumes:
      - cvat_share:/home/django/share:ro
...
volumes:
...
  cvat_share:
    driver_opts:
      type: none
      device: <shared_folder>
      o: bind
...
```
where `...` means skip to next relevant section.

- See the CVAT documentation for the next step as there are numerous ways to do the following (e.g. use  `export `, `env.list` or `.env`, however we recommend the following for access to webui from other machines:
- Create a `.env` file in the directory where your `docker-compose.yml` is located, open it and add these lines:
`CVAT_HOST=<your-ip-address>`
`CVAT_BASE_URL=http://<your-ip-address>:8080/api/v1/`
inserting your machines IP number instead of `<your-ip-address>`.

- Bring up CVAT by running `docker-compose up -d` (skip -d first time to easily see all output to check everything is in order).
- Create CVAT superuser and set password (note: not same command as newer versions):
`docker exec -it cvat bash -ic 'python3 ~/manage.py createsuperuser'`
set administrator name (e.g. admin), mail address and password.
- If you now try to login, CVAT will not allow you to do so for some reason. Bring down CVAT by pressing Ctrl+C in your terminal (if you ran without the -d) or run `docker-compose down`. Then start it all again, `docker-compose up -d`, and everything should be OK.
- Use a browser to visit you CVAT installation on your machine through its IP and login as the administrator user you created and you are off to the races!

### Additional info
- More detailed documenation here: https://github.com/opencv/cvat - Just remember to choose the correct version tag at github.com to ensure that you read the documentation for your choosen version instead of for the latest development version.

- During our project we ran CVAT in a virtual machine with a quite small harddrive. Since CVAT stores a lot of internal information ("thumbnails" of images etc) it might make sense to map some additional storge folders from the container out onto a secondary disk (our was a NFS mount of a file server export). To do this, modify `docker-compose.yml` according to this:
```json
services:
  cvat_db:
...
    volumes:
      - /fs/cvatdata/db:/var/lib/postgresql/data
...
  cvat:
...
    volumes:
      - /fs/cvatdata/data:/home/django/data
      - /fs/cvatdata/keys:/home/django/keys
      - /fs/cvatdata/logs:/home/django/logs
      - /fs/cvatdata/models:/home/django/models
...
volumes:
#  cvat_db:
#  cvat_data:
#  cvat_keys:
#  cvat_logs:
...
```
where `...` means skip to next relevant section. In this example `/fs/cvatdata` was our mount point for the additional harddrive (e.g. NFS mount). Then create directories under your mount point, e.g. `/fs/cvatdata` by:
```
cd /your/folder/
mkdir data db keys logs models
chown <username>:<usergroup> data db keys logs models
```
for the user which will start docker.

- Keep in mind that if you ave already used a newer CVAT version this might be needed: `docker-compose down -v` in the cvat folder. **It will remove the volumes associated with cvat, so beware.** Make sure to back up your annotations before.

