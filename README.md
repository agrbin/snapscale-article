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

We didn't define a goal to help people lose weight, instead our goal is to help people be more aware. The awareness goal is a smaller, prerequisite goal for the greater goal - human health.

Taking a weight measurement has amplified awareness effect if the number is logged, and if you can compare it with last week, or last month. It's the trend that matters, not day-to-day oscillations.

### How

Smart scales like FitBit Aria are a great way achieve this. With [SnapScale](https://snapscale.life) we are giving the same product to people that currently don't own a smart scale. We are not saying that the app doesn't work with smart scales, it works with any scale that shows the measurement to the user!

The weigh-in habit can be measured with two metrics:
- Per-user percentage of days with a measurement (we want to maximize)
- Per-user time spent in the App (we want to minimize)

SnapScale is a simple application that has 3 screens:
- submit a new weight-in measurement that launches a camera
- view your previous measurements
- settings screen that optionally sets up FitBit export and no-weigh-in reminders

We intentionally reduced all clutter that could waste user's time from the app. There is no login to start using the app - login is required later only if you want to export data to FitBit.

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

My time in investigating and trying out existing solutions was roughly spent like this:

- `30%` Classical computer vision techniques (OpenCV and similar)
- `30%` SVHN dataset
- `2%` Google Cloud Vision API
- `38%` Tensorflow custom solutions and transfer learning

I had to implement 5-6 different pipelines that will take my images and labels and convert it to a format that the solutions above take :) That's a lot of hours spent debugging bugs in geometry and image processing code.

I didn't expect to get in a situation where I'll be (re)training new models. Especially because I had only 400 images.

#### Iterations with custom-trained object detection

From their documentation, Google Cloud AutoML Vision Object Detection looks too good to be true. The Web UI enables you to upload images, train the model and see evaluation results. The trainer uses transfer learning which enables it to train something useful even with a few hundred images only.

(image)

Naive as I am, I started by training a model that solves everything at once. The input is the raw user image, and the output are labeled digit rectangles.

This hit two problems:

1) the 7-segment digits sometimes change meaning with rotation (think 2 and 5)
1) the digit was sometimes too small on the original image for the model to pick up

The rotation problem is amplified by the fact that the mobile phone doesn't know how to orientat the image of something that is on the floor.

I decided to split the problem to two phases: detecting the display first, and then detecting the digits in the display. Each phase can be evaluated on its own, which is useful in loss analysis.

#### Display detection

My first attempt was predicting a single rectangle of a single class `display` on the input image. The training set was consisting of upright rotated images labeled with a single rectangle around the display. The AutoML Object Detection learned this with very good precision. At prediction time I would rotate the image 4 times and find the maximum confidence score for the display detection. I assumed that the maximum confidence would be achieved when the display is in upright rotation, just as it was in the training set.

(image, upright detector evaluated on differently rotated images)

Lol, no. I violated the property that training input distribution needs be aligned with the runtime input distribution. The model training saw only correctly rotated images, so the confidences that I was getting for the incorretly rotated images were rubbish - sometimes 0, sometimes 1.

To fix this, I made the problem harder for the model. I expanded each image in the input to `K=4` new images with different rotations. Rotations are done in `360 / K = 90` increments, starting from the upright image. The display label on the upright image became `display_rotated_0_degrees`. On the next image it became `display_rotated_90_degrees`, then `display_rotated_180_degrees` and `display_rotated_270_degrees`. So the number of input images and output classes grew `K` times.

(image, K-rotation input)

During the prediction I would show the input image to the model. The output is the display location and how much it was rotated from the upward position.

(image, K-rotation output)

This worked much better, but it still had some precission losses. The final slam dunk for display+rotation detection was the voting system during prediction. Because the training phase recieved `K` rotations of each image, now it became OK to present the model with `K` rotations of the input image during prediction time. Each of the `K` rotations would give some answer to where is the model and how it's rotated.

If the model was perfect, all evaluations would show the same location and rotation hints would yield different value (`0`, `90`, `180`, `270`) for each rotation. Subtracting model's rotation hint from the image rotation that was used we give the final answer for the expected rotation. Implementing a voting scheme here gave surprisingly good results.

(image, voting)

Note that the hyper-parameter `K` doesn't need to be `4`. If the model is perfect, the maximum angle distance of the hinted rotation and the correct rotation is `(360 / K) / 2`. If `K` is bigger, the display detection problem is more difficult. On the other hand, the outputs are closer to being upright rotated, which makes the digit detection problem less difficult.

(image, comparison between two K values)

#### Digit detection

Digit detection inputs an image and outputs rectangles labeled as digits: `digit_{0-9}`. I expected this to be simple for the AutoML Object Detection. Lol, no. Nothing is simple.

I got many precision losses which I couldn't easily explain.

(image)

Looking through the open sourced `object_detection` Tensorflow on GitHub, which I suspected AutoML was using, I found a parameter
[aug_rand_flip](https://github.com/tensorflow/models/blob/master/official/vision/detection/dataloader/shapemask_parser.py#L131). Based on the docs, the parameter expands input training images with their horizontal flips. This parameter can't be configured in AutoML Object Detection, but if they set it to true it would surely screw up 7-segment digit object detection.

(image)

Google Cloud support answered my question on [cloud-vision-discuss thread](https://groups.google.com/forum/?utm_medium=email&utm_source=footer#!msg/cloud-vision-discuss/6mrbUdWcVys/Tv4nXRy8BAAJ), which helped me understand that AI Platform Object Detection and AutoML Object Detection are quite different.

I submitted a training job to AI Platform (after building yet another pipeline to transform my data), and got excellent results. This was another surprise, because my training set was less than 400 images, and the AI Platform Object Detection trains from scratch - it doesn't do transfer learning.

## Conclusion

### ML principles that I break, and I wish I hadn't

### ML principles that I challenge

### Parting words

On Monday, 2020-10-28 I was in a room in downtown Manhattan packed with excited engineers and PMs who were ready to take over the world. This was AWS Data Lake Day in AWS Loft. I clearly remember a specific sentence - the speaker said confidently:

> You've gotta use ML! If you don't chances are that your competitors will.

I hated it. I was full with disagreement. On the other hand, in the audience, I saw ripples of tough-looking head-nodding experts getting high on agreement with the speaker.

That is when I decided to embrace a new personal engineering principle - "Don't go AI first - always stay user first."

ML is nothing but a tool.

Even The Doors knew this when they played The Roadhouse Blues. They say "Yeah, keep your eyes on the **road**, your hand upon the **wheel**.".  For **road**, they probably meant **user goal** and for the **whell** they meant **the tools** - whatever tools are needed - a for-loop, a compiler, a database or an ML model.

By the way this song is full of good engineering analogies!
