## A brief discussion on Video Analytics (Rough Draft and need your Inputs)

I and two of my collegues **Sachin Chandra** and **Vikash Challa** have been working on Action recognition for quite some time (2 months). We want to compile our experiences and discuss what we have so far learned in this space. I will be basically discussing the following details

- Business Aspect: Why we need Video Analytics?
- Datasets: Openly Available datasets and pitfalls surrounding it. Things we need to consider for annotating our own datasets.
- Buidling data pipelines for Automatic annotation.
- Reasearch papers: Different techiques used for Action Classification
    - C3D networks
    - Spatio-Temporal Neworks (RGB+ C3D FLOW)
    - Kinetics-i3d network (C3D RGB+ C3D FLOW)
    - TSN (Segment Networks)
- Why Action Localization is important ?
- Is Unsupervised Learning a thing?
- Computational cost and other factors to consider when doing Video Analytics


When we are given some grant to work on Video Analytics space, We have taken it with a pinch of salt. This is not typical Image classification or recognition problem where we are now seeing state of the results (Densenet 5.1% error rate  and YOLOV2 is able recognize 9000 objects accurately). Google Deepmind paper **Kinetics-i3d** is trained on 4,00,000 annotated videos across 400 classes (Actions) and still was able to achieve 30.4 mAP score (Activity Net challege). We will later see why this is hard to achieve but I have mentioned this much early to set the expectation.


## Business Impact and Why companies need video analytics?
- A consumer product company wants to know how people are interacting with their product?
    - How much time are they using it for?
    - How comfartable are they using it ? etc
- A retail space in your city might want to see if there is any suspicious activity inside their zone?
- Survillence systems monitoring people

## Datasets:
The following datasets are freely available
- UCF-101
- HMDB-51
- Sports 1 million
- Kinetics
- Youtube 8 million
- Activity Net challege (Untrimmed Test Video dataset)
- Facebook SLAC (Still not released)

All the video datasets are trimmed (10 secs) and labelled accordingly. Each one of them follow their own annotation techinque. But we want to raise some concerns when you are gathering data (Video specifically) for your own task.

- Each class should have unique videos (collected across different backgrounds and people). Redundancy is a alarming problem in the video analytics space.
  - Collecting two video chucks from the same household is a common mistake most people make. This is highly redundant as both the videos are moslty same.
  - Also collect videos with unique backgrounds (Indoor and outdoor, night light vs day light etc).


## Building Data Piplelines
For dividing the dataset into train, val and test, take the following things into consideration
- Make sure your train, val and test dataset are unique. For example, If you are dealing with some household data, make sure that the same houshold data is not present in train and test.
- Incase if you are dealing with only one household data (A mall), Use time to split your data.
- If there is less data available, Use external data from the above mentioned datasets.

Pipleline to Annotate the dataset:
We basically followed the following pipeline to process a video dataset:
- List down all the actions required to recognize in the dataset
- Check if any pre-trained model is available or not in these specific set of actions or not. (In our case yes). Even partially is also fine.
- Extract the part of video where humans are present (Use R-CNN or Mask RCNN etc)
- Send the videos through the pre-trained model and annotate those videos which have high confidence.
- Manually Annotate the videos which doesn't have high confidence.
- Remove Duplicate videos using cosine-similarity (Bottleneck features).

In our case we used,
- Kinetics-i3d model for video annotation and Mask R-CNN to extract video chucks where human is present.
- This reduced our video annotation time by 5x and reduced our human error rate by more than 20% for annotating.

Do some sanity checks
- Unique videos in each class.
- Unique videos in train, val and test.

In some cases, before depolying the model, collect a dataset unianimously from a different set and test it on it


Adding External dataset:
- As Andrew NG explained, since our dataset is very low, we have taken videos from Kinetics and UCF-101 and added them to our train set, keeping our validation and test set videos limited to our own dataset.
- We used 80% - 10% - 10% split

## Building models:
1) We have tried many different models on the action dataset and Kinetics-i3d and TSN performed well for our particular task. Unlike Image dataset, From Videos we need to extract both the action part and object part, We will term them as Flow and RGB

### C3D
Facebook Research has first trained C3D models on UCF-101 and got 70% accuracy on test-set. C3D are nothing but inflating 2D convolutions in 3D (Z space). They have taken 72 frames with stride 32 and trained a model.

- One of the pitfalls is, it uses extremely many parameters and has high possibility to overfit the entire model.
- The Z-axis data is highly redundant. (Consecutive frames contain the same data)


### Spatio-Temporal Networks:
- Uses both RGB and flow Images
- The flow images are generated in the following way
    - Take an Image from the video.
    - Use Image Difference using optical flow (Installing opencv cuda verison is a mess) and calculate the optical flows for next D frames and previous D-frames.
- The network takes one RGB image and 2D frames as input using parallel nets ( Without Weight sharing) and later fuse it using different settings.

This worked well for them and improved the accuracy.

### Kinetics-i3d
- They used both RGB and Flow as 3D images
- Used bn-inception model and inflated them along the z-direction to make use of pre-trained model
- Got 98% accuracy on UCF-101 model and 72% accuracy on Kinetics dataset. Top-5 accuracy is standing at 90%

### TSN
- Generate RGB, Flow_x and Flow_y for the video (10 sec).
- Select 25 images (RGB- time dependent) and 25 flow (flow_x+flow_y, time dependent) and pass through the segmentor network (bninception in parallel and different for both rgb and flow).
- 98% accuracy on UCF-101 model and 76% accuracy on Kinetics dataset. Top-5 accuracy is standing at 92%

PS: Accuracies on these datasets should be taken with a pinch of salt as these are evalutated on trimmed data. We need to test these models on Untrimmed data and calculate mAP as the metric.

## Will action Localization benift?
- We have not check the action Tublet papers in our research but used Action Localization. Fundamentally one of the problem we faced includes
  - RGB images are finding irrelvant features (Humans out of focus) when considering the entire frame.

- So inorder to tackle this we have used Mask R-CNN to indentify human in the video frames and later constructed a Tube using this and extracted the part of video where human is present. This helped us in improving the accuracy.

- We should have specifically used object (human) tracking in this case and used tublets accross a video. We haven't done this as our video set contains 98% of the time single human. We will later explore this space.


## Is Unsupervised training a thing?
- One of the Questions which came to us is "Can you automatically tell us how many unique actions are present in the data?". This is a valid question we thougth of tackling it using the following procedure

- Use a pre-trained model and extract Bottleneck features. (May be train Auto-encoders using video)
- Use T-sne to reduce the dimensionality
- Use K-means and find the optimal number of clusters

We used UCF-101 and extracted the Bottleneck features, Our k-means algoirthm has given the optimal number of clusters to be 92 and when we visualized the dataset similar videos grouped together. We are still not confident of this process and training Auto-encoders is becoming difficult with our settings. We will build this stack as we proceed further.

End notes:
----------
TODO
