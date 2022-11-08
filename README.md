# High level description of the OBDB project repos and how to use them

## Overview

### General info on 4k vs full hd (1920x1080)
- The GoPro cameras used in the project record video in 4k. Due to lens distortion, perspectives and area of interest a full hd part of the image was choosen to be of largest interest. That means the lower half and the middle 2/4 of the image is used. There are still annotations covering the 4k part outside the full hd frame so annotations are automatically cropped if they cross the 4k to full hd boundary. In training we use full hd image.

### Weed_training
- https://github.com/edl78/weed_training
- Main PyTorch training repo for weed detection

### Weed_annotations
- https://github.com/edl78/weed_annotations
- Statistics and database. Has interface to cvat for fetching annotations and inserting them into the MongoDB. Also a mongoExpress web interface to MongoDB for easy access to the database.

### Analytics
- https://github.com/edl78/analytics
- T-sne clustering of bboxes for each task. GPU accelerated via Rapids lib and accessed via an http-api made with flask.
- The Rapids library is only available in conda if we do not like to build from source. This gives double virtualization since we also use Docker for every function/repo.
- The Analytics conda part is notoriously hard to build due to packages being deprecated very fast. Expect to debug and work hard to get it to build. Build failure logs are also very hard to decrypt.



### dev_tips
- https://github.com/edl78/dev_tips
- Short description of how we setup Docker container remote development with vscode. Very handy when in times of pandemics working from home with office servers.


## Workflow

### Getting the data
- Add script to download data with wget...
- Artefacts folder contains a pretrained resnet18 Pytorch model and json files to import to MongoDB for the short path to training. Json files are meta.json, tasks.json and annotation_data.json.

### Getting the code
- git clone repos listed above.

### Building all Docker images
- Build the Docker images by following the respective repos build instuctions.

### Start services
- How to start all services needed is covered in each repo.

### Build and start CVAT
- Follow docs here: https://github.com/openvinotoolkit/cvat
- Use a version with API 1.0 support. For example 1.7.0
- We map a storge folder like this and expect the cvat root to be called /fielddata as this is assumed in the upload code to the http-api.
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



## Short vs long path

### Short path to training
- Build weed_annotations and run as per instructions in the repos. Build weed_training but do not run yet.
- no need to setup cvat for this route.
- load annotations into mongo with mongo interface. Install the MongoDB Database Tools by downloading from mongodb website and follow install instructions. Find the mongodb json files in the artefacts folder downloaded from the OBDB site. To import data into MongoDB use (fill in your username, password and port):
- for the annotation data: `mongoimport --username= --password= --host=localhost --port= --collection=annotation_data --db=annotations annotation_data.json`
- for the meta data: 
`mongoimport --username= --password= --host=localhost --port= --collection=meta --db=annotations meta.json`
- for the tasks data:
`mongoimport --username= --password= --host=localhost --port= --collection=tasks --db=annotations tasks.json`

- use premade pd_train.pkl and pd_val.pkl
- do not care about about the analytics service
- start training as per instructions in the weed_training repo.
- get metrics, see instructions in the weed_training repo.

### Long path to training
- set up all services including CVAT (analytics only for data insight, takes some time to run, in the order of days and we will supply pre calculated T-SNE graphs, so it can be skipped) 
- upload tasks to cvat, validation pickle: `python3 auto_annotate.py --upload_pkl True -p /train/pickled_weed/pd_val.pkl`
- and train pickle: `python3 auto_annotate.py --upload_pkl True -p /train/pickled_weed/pd_train.pkl`
- fetch all annotations via dashboard to mongodb: press update database button in the dashboard running on localhost:8050
- start training `python3 torch_model_runner.py`
- get metrics `python3 test_model.py`


### Finding new optimal hyperparameters
- This requires manual hacking for now.
- In the weed_training repo under code/weed_data_class_od.py, hardcode the return value of `def __len__(self)` to a small value and a multiple of the batch size, for example 4. Run the overfit version of the training, model_trainer_overfit.py by commenting in the import in torch_model_runner.py then run an extensive search with Sherpa by setting the `"run_hpo": 1` to 1 in the json config settings_file_gt_train_val.json. Watch the Sherpa dashbord at localhost:8880 and see the Bayesian optimization at work. The optimal parameters will be saved in /train/resnet18_settings.json if resnet18 was chosen as model. These settings can then be transfered to the config file manually or you can point to it in the torch_model_runner.py training code like this: `trainer.train_model(settings_file='/train/resnet18_settings.json')` 

### Just want to play?
- use the pretrained resnet18.pth with supplied class_map so you know what class predictions mean and implement an inference solution of your choice.