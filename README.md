## Introduction 

I started drilling on SnapScale in mid-2018 with a goal of **creating a weigh-in habit** for a friend of mine. The idea is to convert an old school scale into a smart **using phone's camera** and OCR. The users don't need to look at the number in that moment, the phone does the job and writes the number to the log, optionally user's FitBit profile.

If you're looking for a demo - it's here - [SnapScale - weigh in daily](https://snapscale.life).

This report gives a timeline of a full-stack project using the "sexy" ML tools. Spoiler - there is nothing sexy about it :) the usual engineering tasks are dominating the timeline.

The sexiness and rushing into ML only prolonged the project.

### The goal of this report

I am writing this report to share my thoughts about:

- ML principles for real end-to-end applications
- The state of ML tools today available on Google Cloud mostly
- Rotation-invariant algorithm to detect a display and read out the digits

### Targeted audience

Software Engineers, Data Scientists, Product Managers and other Rock and Roll people.

Less likely, if you are using [the photo weigh-in application](https://snapscale.life), here you can learn how the AI behind it works.

## Case study: a weigh-in habit app, SnapScale 

### Why

The goal of SnapScale is to instill a periodic weigh-in habbit in humans.

A periodic - for example, daily - body weight measurement is already a good diet on its own.

I didn't define a goal to help people lose weight, instead our goal is to help people be more aware. The awareness goal is a smaller, prerequisite goal for the greater goal - human health.

Taking a weight measurement has amplified awareness effect if the number is logged, and if you can compare it with last week, or last month. It's the trend that matters, not day-to-day oscillations.

### How

Smart scales like FitBit Aria are a great way achieve this. With [SnapScale](https://snapscale.life) I am giving the same product to people that currently don't own a smart scale. I am not saying that the app doesn't work with smart scales, it works with any scale that shows the measurement to the user!

The weigh-in habit can be measured with two metrics:
- Per-user percentage of days with a measurement (we want to maximize)
- Per-user time spent in the App (we want to minimize)

SnapScale is a simple application that has 3 screens:
- submit a new weight-in measurement that launches a camera
- view your previous measurements
- settings screen that optionally sets up FitBit export and no-weigh-in reminders

I intentionally reduced all clutter that could waste user's time from the app. There is no login to start using the app - login is required later only if you want to export data to FitBit.

## Implementation details worth sharing

### AI Last

Engineering-wise, the central and most "sexy" part of the application is the component that reads the weight measurement from the image. But, before diving into that part - I defined and implemented everything else first:

- the complete mobile Application including photo-based weight-logging, reminders, and export to FitBit.
- launching the application with a fake backend - each image would trigger a notification on my phone for me to read the numbers instead of the ML model
- implementing the UI for reading the numbers from images and labeling them
- privacy policy around images that are uploaded to the backend
- iterating on the usability of application
- defining the evaluation metrics for the image recognition subproblem
- implementing the evaluation framework for image recognition

Only then, when everything was ready and it seemed that the product may work, I tackled the AI:
- picking the technology and implementing the image recognition solution

This is what I mean by AI comes last. Even if using an existing OCR was successful, I would still do all the same steps. The existing OCR library would still have to answer to the same requests and fulfill the same metrics.

By the time I started iterating on the model, I had seen ~450 user images and had a good idea of all kinds of corner cases that might happen. I also had a dataset of images, defined metrics, debugging frontend and evaluation framework in place.

One risk of this approach is betting everything on the fact that the last step (AI) will work. It could've happened that OCR at the end would be too difficult to implement, and the project would fall apart.

### The app

The application is built using [Expo](https://expo.io) framework which is a wrapper around [React Native](https://reactnative.dev/).  It's suitable for fast prototyping which is exactly what I needed. Same codebase is executed on both platforms. If Expo would have a book it would be called "App development for dummies".

There is really nothing interesting in the client code implementation here.

### Backend

The app needs a backend because the logs and images are stored serverside. There is no authentication of any kind, but there is a lightweight register step.

On app install, user needs to agree on terms of use and sign using a captcha. The successful captcha challenge gives to the application a session token that is later required to communicate with the server. This was a good way to design an application + backend storage without requiring a login.

Although login sounds like a good alternative, I suspected it would repel some users due to sensitivity of the images that are being sent.

For a backend, I deployed a [NodeJS](https://nodejs.org) application on [Google App Engine](https://cloud.google.com/appengine).  To get a free-of-charge deployment for prototypes with low amount of requests, the trick is to use `standard` environment with `F1` instance clasee. The `app.yaml` config looks like this:

```
runtime: nodejs12
env: standard
instance_class: F1
```

The database for metadata used is [Google Cloud Datastore](https://cloud.google.com/datastore) and for image store I used [Google Cloud Storage](https://cloud.google.com/storage). Both systems are free-of-charge for prototypes with low amount of requests.

The natural opportunity here is to move the model execution to the client, which will remove the need for backend, and simplify privacy policy.

### Labeling UI

When a new image is submitted to the system, and the model is not cofident about the results (or when the model was not even there yet) an IFTTT notification would hit my phone with the link to the Labeling UI. The Labeling UI would show the image, let me translate and rotate it, and label the digits. The digits are labeled with rectangles that all share y coordinates, and all have the same dimensions. This is possible because prior to labeling the digits, the image is rotated in such way that the display is upright.

(image)

Note that I made an assumption that the digits are monospace. Well :) Assumption turns out to be the mother of all fuckups.

(image)

I manually processed ~450 images so far. The Labeling UI is implemented in HTML5, mainly using Canvas API in JavaScript. The image transformation is encoded using the affine transformation matrix, [DOMMatrix](https://developer.mozilla.org/en-US/docs/Web/API/DOMMatrix).

The images without visible weight measurement are discarded. All other images are labeled and split into training/test/validation with 70/10/20 ratios.

Having only good images in the dataset, to evaluate the algorithm I defined the accuracy metric as follows:

- **accuracy**: percentage of images in which all digits are correctly identified

If an image that shows `78.2` gets recognized as `78.3`, the accuracy is not satifsfied.

### Digit recognition

In this moment, ~2 months in the project I had an application that works, with 1-2 requests per day and I was ready to implement the digit recognition submodule. I expected that I will find something that works out of the box, and that most of engineering will be just plugging it in and evaluating accuracy. I didn't expect that I'll be trainig new models.

#### Related work

Classical computer vision techniques:

* [https://github.com/jacnel/wetr](https://github.com/jacnel/wetr)
* [https://github.com/NatholBMX/ScaleDigitizer](https://github.com/NatholBMX/ScaleDigitizer)
* [https://github.com/charleswest/Weight](https://github.com/charleswest/Weight)
* [https://github.com/suyashkumar/seven-segment-ocr](https://github.com/suyashkumar/seven-segment-ocr)
* [https://github.com/kazmiekr/GasPumpOCR.git](https://github.com/kazmiekr/GasPumpOCR.git)
* [https://github.com/bikz05/digit-recognition](https://github.com/bikz05/digit-recognition)
* [https://github.com/g-sari/pyautodigits](https://github.com/g-sari/pyautodigits)
* [http://www.dccia.ua.es/~sco/ComputerVision/embedded.pdf](http://www.dccia.ua.es/~sco/ComputerVision/embedded.pdf)
* [https://hackernoon.com/building-a-gas-pump-scanner-with-opencv-python-ios-116fe6c9ae8b)](https://hackernoon.com/building-a-gas-pump-scanner-with-opencv-python-ios-116fe6c9ae8b))
* [https://towardsdatascience.com/build-a-multi-digit-detector-with-keras-and-opencv-b97e3cd3b37](https://towardsdatascience.com/build-a-multi-digit-detector-with-keras-and-opencv-b97e3cd3b37)
* [https://towardsdatascience.com/scanned-digits-recognition-using-k-nearest-neighbor-k-nn-d1a1528f0dea](https://towardsdatascience.com/scanned-digits-recognition-using-k-nearest-neighbor-k-nn-d1a1528f0dea)
* [https://www.kurokesu.com/main/2017/02/20/dumb-thermometer-gets-digital-output/](https://www.kurokesu.com/main/2017/02/20/dumb-thermometer-gets-digital-output/)
* [https://www.programcreek.com/python/example/89294/cv2.watershed](https://www.programcreek.com/python/example/89294/cv2.watershed)
* [https://www.pyimagesearch.com/2017/02/13/recognizing-digits-with-opencv-and-python/](https://www.pyimagesearch.com/2017/02/13/recognizing-digits-with-opencv-and-python/)
* [https://www.sciencedirect.com/science/article/pii/S235248471930174X](https://www.sciencedirect.com/science/article/pii/S235248471930174X)

Street View House Number Dataset related solutions:

* [https://medium.com/project-agi/getting-started-with-street-view-house-numbers-svhn-dataset-a96e76962504](https://medium.com/project-agi/getting-started-with-street-view-house-numbers-svhn-dataset-a96e76962504)
* [https://arxiv.org/pdf/1312.6082.pdf](https://arxiv.org/pdf/1312.6082.pdf)
* [http://ufldl.stanford.edu/housenumbers/](http://ufldl.stanford.edu/housenumbers/)
* [https://github.com/potterhsu/SVHNClassifier](https://github.com/potterhsu/SVHNClassifier)
* [https://github.com/znat/udacity-digit-recognition-program-svhn/blob/master/project-report/project-report.md](https://github.com/znat/udacity-digit-recognition-program-svhn/blob/master/project-report/project-report.md)

Open Source tensorflow Object Detection pipeline:

* [https://github.com/tensorflow/models/tree/master/research/object_detection](https://github.com/tensorflow/models/tree/master/research/object_detection)

Documentation about Google Cloud object detection solutions as service:

* [https://cloud.google.com/vision/automl/object-detection/docs](https://cloud.google.com/vision/automl/object-detection/docs)
* [https://cloud.google.com/ai-platform/training/docs/algorithms/object-detection-start](https://cloud.google.com/ai-platform/training/docs/algorithms/object-detection-start)

My time in investigating and trying out existing solutions was roughly spent like this:

- `30%` Classical computer vision techniques (OpenCV and similar)
- `30%` SVHN dataset
- `40%` Tensorflow custom solutions and transfer learning

I had to implement 5-6 different pipelines that will take my images and labels and convert it to a format that the solutions above take :) That's a lot of hours spent debugging bugs in geometry and image processing code.

At the end I was left with no options but to train something on my own... but I had only 400 images.

#### Iterations custom object detection solution

Naive as I am, I started by training a model that solves everything at once. The input is the raw user image, and the output are labeled digit rectangles.

This hit two problems:

1) the 7-segment digits sometimes change meaning with rotation (think 2 and 5)
1) the digit was sometimes too small on the original image for the model to pick up

The rotation problem is amplified by the fact that the mobile phone doesn't know how to orientat the image of something that is on the floor.

I decided to split the problem to two phases: detecting the display first, and then detecting the digits in that display. Each phase can be evaluated on its own, which is useful in loss analysis.

#### Display+orientation detection

#### Digit detection

#### Serving

## Conclusion

### ML principles that I break, and I wish I hadn't

### ML principles that I challenge

### Parting words

On Monday, 2020-10-28 I was in a room in downtown Manhattan packed with excited engineers and PMs who were ready to take over the world. This was AWS Data Lake Day in AWS Loft. I clearly remember a specific sentence - the speaker said confidently:

> You've gotta use ML! If you don't chances are that your competitors will.

I hated it. I was full with disagreement. On the other hand, in the audience, I saw ripples of tough-looking head-nodding experts getting high on agreement with the speaker.

That is when I decided to embrace a new personal engineering principle - "Don't go AI first - always stay user first."

ML is nothing but a tool.

I think that even The Doors knew this when they played The Roadhouse Blues. They say "Yeah, keep your eyes on the **road**, your hand upon the **wheel**.".  For **road**, they probably meant **user goal** and for the **whell** they meant **the tools** - whatever tools are needed - a for-loop, a compiler, a database or a model.

If you didn't yet, listen to the song! It is full of good engineering analogies!
