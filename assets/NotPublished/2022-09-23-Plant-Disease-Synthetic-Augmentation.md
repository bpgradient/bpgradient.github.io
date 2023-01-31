---
title: "Generating Synthetic Images To Classify Plant Disease"
toc: true
toc_sticky: true
tags: advanced classification augmentation
categories:
  - DeepLearning
  - Classification
header:
  teaser: /assets/images/benface.png

image: benface.png
---

This idea originated from a discussion that I had with Sam Murcio (brother from another mother) about using ML to identify the early onset of crop disease. This was just one piece of the puzzle, but I don't want to clue anyone else in on his billion dollar idea. ;)

From an application perspective, if we can use ML to automatically identify crop types, detect whether the leaves are diseased or not, and what type of disease (if any) is present, it would be a great help with early remediation efforts. 

Overview
----------------------------------

The short description of this project is:

1. Download the dataset 
2. Perform exploratory analysis on the data
3. Show a method to detect and remove blurry images
4. Train a Convolutional Neural Network (CNN) using the downloaded data
5. Use data augmentation techniques to create more data
6. Show how data augmentation impacts model performance
7. Build/train a Generative Adversarial Network (GAN) to create synthetic copies
8. Show how synthetic data impacts model performance

About The Data
----------------------------------

In this post I'll be using a publicly-available dataset ([found here](https://www.kaggle.com/datasets/vipoooool/new-plant-diseases-dataset)) that has approximately 88,000 images of crop leaves, spanning 38 different classes. 

Among these 38 classes are what I call root classes (such as Apple, Corn, Grape, Potato, etc.) and sub-classes. The subclasses aren't separate, but are instead part of the root class. 

For example, 4 of the 38 classes are related to Apple plants, but show different stages/disease of the leaves. These classes are listed as:

**Apple**:
* Scab
* Rot
* Rust
* Healthy

The data is already already split into training and validation sets for us, with about 70,000 images belonging to the training set and 17,500 for validation. There is an additional post-training test set that includes 33 images for simulating unseen data. This helps in removing steps involved in identifying and splitting the images ourselves.

You'll see this again in the EDA section, but the data is also *mostly* balanced, which means we should be able to work with it as-is and not implement methods to handle imbalanced data.

However, 


AWS Rekognition or Azure Face API?
----------------------------------

<div style="text-align:center"><img src="/assets/images/rekogface.png" /></div>

Both AWS' Rekognition service, and the Azure Face API service, are hosted computer vision services that help with video or image analysis tasks. In typical 'AWS vs Azure' fashion, the services are similar but also different. The **TL;DR**:

- AWS Rekognition does face detection/recognition, text extraction, content moderation, + many other things
- Azure Face API primarily does face detection/recognition 

If you're looking for true comparison I would recommend [Azure Cognitive Services](https://azure.microsoft.com/en-us/services/cognitive-services/#features) in comparison to Rekognition. 

For our need we only focused on face detection/recognition, not features like text extraction or content moderation. 

Understanding the Data Flow
-----------------------------------------

If you aren't very strong with coding/Python, below is some rough pseudocode that describes what the Azure Face API code does in our example. 

* Start
* Initialize variable _face_client_: Face API Auth & URL variables
* Initialize variable _cap_: video capture endpoint (webcam, RTSP, file, etc.)
* Initialize variable _frameNumber_: incrementing video frame
* Start While loop
  - If frameNumber less than 5
    - Call function _standard_stream()_
      - Read video frame as image
      - Convert to grayscale
      - Detect faces and locations in image
      - For each face in image
        - Draw a bounding box around the face
        - If Face API results already exist
          - Add detected sentiment text to image
          - Add confidence score text to image
          - Add detected age text to image
          - Display the augmented frame with inference results
        - If Face API results **do not** exist
          - Display the frame with just bounding box
    - Increase frameNumber counter by 1
  - If frameNumber **not** less than 5
    - Call function _send_inference()_
      - Read video frame as image
      - Convert to grayscale
      - Detect faces and locations in image
      - Convert image to bytes
      - Send bytes to Face API
      - If API call is successful
        - Call function _get_emotion()_ 
          - Parse results (ex. emotion, age, gender, confidence score)
          - Return results in dictionary
        - For each face in image
          - Draw a bounding box around the face
          - Add detected sentiment text to image
          - Add confidence score text to image
          - Add detected age text to image
          - Display the augmented frame with inference results
          - Persist inference results in global variables
      - If API call is **not** successful
        - Display the frame with just bounding box
    - Reset frameNumber to 0
- Release captured frame(s) from memory

From the pseudocode from above you can probably tell that we're only sending every 5th frame to the inference API. We do this for a few reasons:

1. Reduce latency/lag on video stream by not analyzing every frame
2. Reduce cost and potential API transactions per second throttling
3. Analyzing a frame every ~33 milliseconds (if 30 FPS video) might not provide value depending on the use case. Ex. - we don't care to know if someones sentiment changes every 1/30th of a frame.

Give Me the Code!
-----------------------------------------

Rather than posting all the code here and making this post even longer, [here is a link to where the code lives on my GitHub](https://github.com/bcprescott/cloudai/blob/main/Azure/FaceAPI/FaceDetection.py). Please note that you will need to change the following: 

{% highlight python %}

KEY = "xxxxxxxxxxxxxxxxxxxxxx"   <- replace with your Azure Face API endpoint key

ENDPOINT = "https://{replace with your Face API service name}.cognitiveservices.azure.com/"

{% endhighlight %}

Depending on your use case, you may also need to change what OpenCV is using as a video source. 

In my script I have it reading from local system source 0, which will use my default laptop webcam. You could change this to a different number for USB webcams, or even a video feed URL. For an RTSP example:

{% highlight python %}

cap = cv2.VideoCapture('rtsp://32.242.152.88:8554/camera1')

{% endhighlight %}

Reviewing the Results
-----------------------------------------

If all goes well you should have a Python window display that is showing you the frame-by-frame live playback (example below). 

![Playback](/assets/images/frameplayback.gif)

Remember, in this scenario we are reading a video feed, which is just a stream of images/frames. Therefore, we can treat our manipulation and playback as individual images. 

For example, you might save each frame (or every nth frame) to storage. Maybe you are saving inference results (happy/sad, age, confidence, etc.) to a database. When a certain event is recorded it will pull one of the video frames saved to storage and display it for the end user. Who knows, the possibilities are (mostly) endless!

Ethical Considerations
-----------------------------------------

As with any machine learning solution, always consider how it will be used/applied. **Machine learning algorithms are not perfect and should not be treated as such.** 

How many times have you looked at someone and had to do a doubletake because you thought it was someone you knew? How accurate are you in guessing someone's age (within 1 or 2 years)? Personally, I'm terrible at it. I tend to assume someone is much older than they are, no offense :) .

Machine learning is based heavily on statistics/probability, so we should know that it won't be 100% accurate all of the time. Therefore, a ML application _could be_ dangerous depending on how we may react to an algorithm's result.

Facial recognition has gained some bad press due to how it is applied and blindly trusted. For example, police have used facial recognition and it has led to [innocent people being arrested due to false positives](https://www.wired.com/story/wrongful-arrests-ai-derailed-3-mens-lives/). 

Below is a notice from Microsoft on their [Face API documentation pages](https://docs.microsoft.com/en-us/azure/cognitive-services/face/overview)

![MSFTwarn](/assets/images/msftwarning.png)

On another extreme, consider the military's use of face recognition. Incorrectly identifying a person of interest may lead to an innocent person being killed. However, correctly identifying someone quickly could save lives. Something worth noodling on.

There are many scenarios where face detection/recognition are used correctly, where occasional false positives or negatives may be acceptable and wouldn't lead to a life threatening situation. **Remember to always consider the impact a solution may have on the people and environment before implementing it.**