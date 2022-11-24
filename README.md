# High level description of the OBDB project repos and how to use them
You can skip directly to Workflow below to get an idea of what can be done with the OBDB repos. For some more details continue reading here.

## Overview of repos
There are 4 github repos that belong to the project (and this repo just for some documentation).

### dev_tips
Short internal description of how we used to setup Docker container remote development with VScode during the pandemic. Very handy in times of working from home to office servers. Repo with description [here](https://github.com/edl78/dev_tips).

### Weed_training
The [weed_training](https://github.com/edl78/weed_training) repo contians PyTorch code for training a network to do weed detection.

### Weed_annotations
The [weed_annotations](https://github.com/edl78/weed_annotations) repo contains code which sets up a new pipeline for do iterative semi-automatic annotations. It was developed to communicate with [CVAT](https://github.com/opencv/cvat) through the API. This work was done during the API 1.0 time period of CVAT, i.e. CVAT version up to 1.7.0, and is the reason why you will need to use an old version of CVAT (see further instructions).  Of course you could always update the code to support CVAT API 2.0 and if you do please consider sharing your updates back to us!

Has interface to cvat for fetching annotations and inserting them into a MongoDB external from CVAT. Also a mongoExpress web interface to MongoDB for easy access to this database.

### Analytics
The [analytics]( https://github.com/edl78/analytics) repo contains a few helper functions we used during the project to study our data:
- T-sne clustering of bboxes for each task. GPU accelerated via Rapids lib and accessed via an http-api made with flask.
- The Rapids library is only available in conda if we do not like to build from source. This gives double virtualization since we also use Docker for every function/repo.
- The Analytics conda part is notoriously hard to build due to packages being deprecated very fast. Expect to debug and work hard to get it to build. Build failure logs are also very hard to decrypt.


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

For each video file captured by the GoPro we used [ffmpeg]() to extract key frames in sequence. The naming of the images is just a sequence of numbers for the key frames.

### Extraction of frames from video

Keyframes were extracted from the video files using the following call to ffmpeg:

`YYYYYYYYYYYYYYYYYYY`


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
  cvat:
    volumes:
      - cvat_share:/home/django/share:ro

volumes:
  cvat_share:
    driver_opts:
      type: none
      device: <shared_folder>
      o: bind
```

- *We map a storge folder like this in the cvat docker-compose.yml and expect the cvat root to be called fielddata as this is assumed in the upload code to the http-api.*
**Vad betyder detta? Använder vi en annan mappad katalog för att lagra data?? (den ovan är ju read-only)**

- In the cvat folder run: `export CVAT_HOST=<your-ip-address>` (you should maybe set this permanently depending on your use case)
- In `env.list` that is sent to the container, set this (as above replace with your IP address where CVAT runs): `CVAT_BASE_URL=http://<your-ip-address>:8080/api/v1/`

- Bring up CVAT by running `docker-compose up -d` (skip -d first time to easily see all output to chech everything is in order).

- Create CVAT superuser and set password (not same as newer versions): `docker exec -it cvat bash -ic 'python3 ~/manage.py createsuperuser'`

- Use a browser to visit you CVAT installation on your machine through its IP.

### Additional info
- More detailed documenation here: https://github.com/openvinotoolkit/cvat - Just remember to choose the correct version tag so you read the documentation for you choosen version instead of the latest development info.
- Keep in mind that if you ave already used a newer CVAT version this might be needed: `docker-compose down -v` in the cvat folder. **It will remove the volumes associated with cvat, so beware.** Make sure to back up your annotations before.

## Workflow

### Getting the data
- Add script to download data with wget...
- Artefacts folder contains a pretrained resnet18 Pytorch model and json files to import to MongoDB for the short path to training. Json files are meta.json, tasks.json and annotation_data.json.

### Getting the code
- git clone repos listed above.

### Building all Docker images
- Build the Docker images by following the respective repos build instuctions. This applies to weed_training and weed_annotations, analytics is optional, obdb_docs and dev_tips are just for documentation.

### Start services
- How to start all services needed is covered in each repo.

### Clone and start CVAT

This is, as mentiond above, only needed if you like to inspect, modify or add annotations to the dataset. CVAT is not needed if you will follow the fast track to training below.

## Fast-track to training vs complete setup

### Fast-track to training
- Build weed_annotations and run as per instructions in the repos. Build weed_training but do not run yet.
- no need to setup cvat for this route.
- load annotations into mongo with mongo interface. Install the MongoDB Database Tools by downloading from mongodb website and follow install instructions. Find the mongodb json files in the artefacts folder downloaded from the OBDB site. To import data into MongoDB use (fill in your username, password and port):
- Known bugs in the bitnami/mongodb: must initialize with port 27017 and do not change root user name! Otherwise it will not work...
- https://www.mongodb.com/try/download/database-tools
- for the annotation data: `mongoimport --username= --password= --host=localhost --port= --collection=annotation_data --db=annotations annotation_data.json`
- for the meta data:
`mongoimport --username= --password= --host=localhost --port= --collection=meta --db=annotations meta.json`
- for the tasks data:
`mongoimport --username= --password= --host=localhost --port= --collection=tasks --db=annotations tasks.json`

- use premade pd_train.pkl and pd_val.pkl
- do not care about about the analytics service
- start training as per instructions in the weed_training repo.
- get metrics, see instructions in the weed_training repo.

### Complete setup of training and possibility to annotate
- set up all services including CVAT (analytics only for data insight, takes some time to run, in the order of days and we will supply pre calculated T-SNE graphs, so it can be skipped)
- upload tasks to cvat code found in the weed_training repo under code, validation pickle: `python3 auto_annotate.py --upload_pkl True -p /train/pickled_weed/pd_val.pkl`
- and train pickle: `python3 auto_annotate.py --upload_pkl True -p /train/pickled_weed/pd_train.pkl`
- fetch all annotations via dashboard to mongodb: press update database button in the dashboard running on localhost:8050
- start training `python3 torch_model_runner.py`
- get metrics `python3 test_model.py`


### Finding new optimal hyperparameters
- This requires manual hacking for now.
- In the weed_training repo under code/weed_data_class_od.py, hardcode the return value of `def __len__(self)` to a small value and a multiple of the batch size, for example 4. Run the overfit version of the training, model_trainer_overfit.py by commenting in the import in torch_model_runner.py then run an extensive search with Sherpa by setting the `"run_hpo": 1` to 1 in the json config settings_file_gt_train_val.json. Watch the Sherpa dashbord at localhost:8880 and see the Bayesian optimization at work. The optimal parameters will be saved in /train/resnet18_settings.json if resnet18 was chosen as model. These settings can then be transfered to the config file manually or you can point to it in the torch_model_runner.py training code like this: `trainer.train_model(settings_file='/train/resnet18_settings.json')`

### Just want to play?
- use the pretrained resnet18.pth with supplied class_map so you know what class predictions mean and implement an inference solution of your choice.
