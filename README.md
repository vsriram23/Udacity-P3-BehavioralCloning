# Udacity-P3-BehavioralCloning
Submission for Udacity's Project 3 - Training Car to Drive Autonomously in Simulator

# Approach
My general approach for this project was to use the provided training data from Udacity and as many data augmentation techniques as necessary to produce a working model. While this project could have been handled with regression or classification (binning the steering angles into classes) I found regression to be much more successful and easier to implement. I tried two different models: the first being the well-established NVIDIA model from their paper (https://images.nvidia.com/content/tegra/automotive/images/2016/solutions/pdf/end-to-end-dl-using-px.pdf) and the second being a similar model to what I had developed for Udacity’s Open Source Challenge #2 ‘Using Deep Learning to Predict Steering Angles’ (https://github.com/chrisgundling/self-driving-car/tree/master/steering-models/community-models/cg23). The models were vastly different as NVIDIA’s only uses around 250,000 parameters whereas the VGG style model that I first tried had nearly 34 million parameters. 

I experimented with various data-preprocessing techniques, 7 different augmentation methods and varied the dropout of each of the two models that I tested. In the end I found that NVIDIA’s architecture did a better job generalizing to the test Track (Track2) and require less tuning. My final NVIDIA model did not use dropout and was able to drive on both Track 1 and 2. I ran into steering latency issues with the larger VGG model while testing in the simulator on CPU. I ended up scaling down the image size on this model from 128X128 to 64X64 and halving the size of the fully-connected layer. This allowed the VGG style model to drive on both Track 1 and 2. I have included only the NVIDIA model for the reviewer to test as the VGG model was too large to upload. The best results obtained with the NVIDIA model were using a throttle value of 0.2 on Track 1 and 0.3 on Track 2 to get up the steepest hills.

# Hardware/Testing Setup
I trained these models using a modified version of Udacity’s CarND AWS AMI with a g2.2xlarge GPU and tested the models on my Macbook Pro laptop with CPU. The simulator was set to the 'Fastest' setting with screen resolution of 640 X 480. I adjusted the drive.py script to implement the same pre-processing methods that I used for the training data (cropping, re-sizing, normalizing images). The NVIDIA model can be trained using the following command for GPU or CPU:
```
python model.py --dataset data/ --model cnn --nb-epoch 10 --resized-image-height 66 --resized-image-width 200
```
Once the model is trained, copy the trained model file to model.h5 and run the following script to produce the .json file:
```
python print_json.py
```
Finally, test it in the simulator using the command:
```
python drive.py model.json
```

# Data Exploration
The data provided by Udacity consisted of 8036 center, left and right .jpg images for a total data size of 24109 examples (only 323 MB of data!). These images were of 160 Height by 320 Width. An example is shown below of the left/center/right images taken at one timestep from the car.

--------------------------------------------------------------------------------------------------------------
               Left Camera                          Center Camera                        Right Camera                
------------------------------------- ------------------------------------ -----------------------------------
![alt tag](images/LeftRightCenter.png)

The steering angles provided in the driving log have been plotted below. 
![alt tag](images/UdacitySteering.png)

The steering angles in the driving_log.csv file came pre-scaled by a factor of 1/25 so that they are between -1 and 1 (the max/min steering angle produced by the simulator is +/- 25 degrees). I knew that I would need an evaluation metric for the model training, so I split out 480 center camera images and their corresponding steering angles to use for the validation set. This is a smaller validation set (validation set size approximately 5% of the center camera images used in training set) than I would typically use, but I didn’t want to give up too much of the training data. I could have implemented k-fold cross validation to develop a better validation error metric, but I found that the single validation set corresponded fairly well with actual driving capability. I did not use a dedicated “test set” to calculate the RMSE, but I instead considered the performance on Track 2 to be my test set. I held off on testing on Track 2 until I found a model that performed very well on Track 1.

# Data Pre-Processing
I followed the discussions closely on Slack regarding collecting training data. It seemed that a lot of people were having difficulties collecting “smooth” steering data using just their keyboards. The Udacity data helped this problem some, but based on the experience that I had will using real human steering data from Udacity’s Challenge #2, the provided simulator steering data still appeared much more step like. 

## Exponential Smoothing
I addressed this issue by applying an exponential smoothing to the steering training data in the driving_log.csv file. Exponential smoothing (Brown’s Method) helped to produce smoother steering transitions, but also truncated and shifted the data. I applied a scaling to recover some of the amplitude of the steering and shifted the data by several time steps so that the steering would not be delayed. The result of the exponential smoothing can be seen below for the first ~1200 training examples.

 <img src="images/Smoothing.png" width="800">

## Normalization
Similar to Project 2, I normalized all of the image pixel values using the equation (x-128)/128. This normalizes the values to be between -1 and 1. I did this for the training data, validation data and implemented it in the drive.py script for testing the model.
## Cropping
I cropped off the bottom 20 pixels and the top 40 pixels from each image after augmentation. This removed the front of the car and most of the sky above the horizon from the images. I did this for the training data, validation data and implemented it in the drive.py script for testing the model.
## Re-sizing
For the NVIDIA model, I stayed consistent with their approach and used images with a height of 66 and width of 200 pixels. For the VGG style model I used 64 x 64 images after scaling down the orignal model from Udacity’s Challenge #2. I did this for the training data, validation data and implemented it in the drive.py script for testing the model.
## Data Augmentation
I used 7 different augmentation techniques to increase the number of images that the model would see during training. This significantly reduced my model's tendency to over-fit the data to the point where I saw no performance benefit of using dropout with the NVIDIA model. The augmentations were performed on the fly using a custom generator/yield method. The data augmentation techniques that I implemented were the following:

1.	Perspective/Viewpoint Transformations – Similar to what is described in the NVIDIA paper, I applied rotational perspective transforms to the images. The function to perform this operation was provided by Udacity’s Yousuf Fauzan after I met him at the “Outside the Lines” event as he had already implemented this technique for Challenge #2. The script uses a rotation matrix to do a perspective transform of varying degree to the image. I had to do some tuning to the steering angle adjustment for the perspective transforms as I found that a one-to-one perspective transform angle to steering angle adjustment was much too large. I settled on rotating the images uniformly between -80 to 80 degrees. I then divided the perspective angle by 200 in order to adjust the steering angle. This gave max/min steering angle adjustments of +/- 0.4 units or 10 degrees.

2.	Image flipping – Since the left and right turns in the training data are not even, image flipping was important for model generalization to Track 2. I also flipped the sign of the steering angle when an image was flipped.

3.	Left/right camera images – I used the left/right camera images from the car which immediately triples the training data size. After closely examining the left/right images and looking for common features to the center images, I estimated that the left right images where offset horizontally from the center camera by approximately 60 pixels. Based on this information, I chose a steering angle correction of +/- 0.25 units or +/- 6.25 degrees for these left/right images. 
 - I also tried to implement a speed based steering angle correction since my intuition was that at higher speeds the steering correction should be smaller or more gradual. I was surprised to find that I could not get this to work as well as having a constant steering angle correction. I think that with further adjustment and better knowledge of the left/right camera image location that this method would work.
 - Speed based steering adjustment was implemented by defining a response time of 2 seconds for the car to return to center. As the speed of the car increases, the steering angle needed to return to center in 2 seconds decreases. The following diagram then shows how the steering angle correction was calculated:
 
 <img src="images/SpeedAdjust.png" width="300">

4.	Horizontal/Vertical shifting – I applied horizontal and vertical shifting to the images. My max/min horizontal and vertical shifts were 40 pixels in each direction. I tuned this value during model training. Considering that I estimated the left/right images to be offset by 60 pixels, I applied a slightly smaller steering angle correction for the horizontal shifts. The vertical shifts had no steering angle correction. I also attempted speed based correction for the horizontal shifts. 

5.	Image Brightness – I adjusted the brightness of the images by converting to HSV color space and scaling the V pixels values from 0.5 to 1.1. This was mostly to help generalize to Track 2 where the images were darker in general.

6.	Image Blurring – I’m not sure how useful this technique was for the simulator, but this technique should help the model generalize when using more “real world” type data that does suffer from blurring at times. I used a variable Gaussian smoothing to blur the images.

7.	Image Rotations – Different from perspective transforms, I implemented small rotations to the images to simulate jittering of the camera. Once again, not sure how useful this was for the simulator, but would be useful for a real camera on a self-driving car.

# Data Generator
The implemented data generator selected randomly between the center/left/right images and also selected at random which augmentation techniques to apply. I found that only providing augmented training data did not work as well as training the model with a combination of the non-augmented original images and the augmented images. I also implemented a bias towards first training the model with larger turns and then allowing the data with smaller turns to slowly leak into the training. This idea is directly credited to a few of the other students who posted this idea on Slack. If the model is initially trained with low steering angles it will be biased towards straighter driving and I found that it did not perform well in the corners. The generator also shuffled the training data, whereas the validation data was read in in sequence. Using the data generator, a figure is shown below that gives the images and augmentation choices that the data generator produces for the first 30 training examples during a training run. The image titles are given as follows:

- ang: Steering angle label for image
- cam: Camera selection (left,center,right)
- aug: 1 is no augmentation, 2 is yes augmentation
- opt: Data augmentation option is: (1. Flip, Jitter, Blur, Brightness), (2. Shift Image), (3. Rotational Viewpoint Transform)

<img src="images/Augmentations2.png" width="2000">

# Model Setup and Hyper Parameters
My goal was to train each of my two models 1. NVIDIA type and 2. VGG type with as many similar hyper-parameters as possible. I used the following parameters for training of both models.

- Max number of Epochs – 10 (5 or 6 Epochs of training typically gave best model for NVIDIA and only 1-3 Epochs for VGG style)
- Samples Per Epoch – 21632
- Batch Size - 64
- Optimizer - Adam with learning rate 1e-4
- Activations - Relu for VGG style and Elu for NVIDIA model

The NVIDIA model implemented in Keras is shown below:
```
# Layer 1
x = Convolution2D(24, 5, 5, activation='elu', subsample=(2, 2), border_mode='valid', init='he_normal')(img_input)

# Layer 2
x = Convolution2D(36, 5, 5, activation='elu', subsample=(2, 2), border_mode='valid', init='he_normal')(x)

# Layer 3
x = Convolution2D(48, 5, 5, activation='elu', subsample=(2, 2), border_mode='valid', init='he_normal')(x)

# Layer 4
x = Convolution2D(64, 3, 3, activation='elu', subsample=(1, 1), border_mode='valid', init='he_normal')(x)

# Layer 5
x = Convolution2D(64, 3, 3, activation='elu', subsample=(1, 1), border_mode='valid', init='he_normal')(x)

# Flatten
y = Flatten()(x)

# FC 1
y = Dense(100, activation='elu', init='he_normal')(y)

# FC 2
y = Dense(50, activation='elu', init='he_normal')(y)

# FC 3
y = Dense(10, activation='elu', init='he_normal')(y)

# Output Layer
y = Dense(1, init='he_normal')(y)

model = Model(input=img_input, output=y)
model.compile(optimizer=Adam(lr=1e-4), loss = 'mse')
```

# Model Architecture
The model architectures for each of the two models can be seen below. As mentioned in the introduction, my main tuning parameter with these models was the dropout. For the NVIDIA model it was somewhat surprising to find that the model performed best on both Track 1 and 2 with no dropout. The model size is relatively small and with all of the applied image augmentations any dropout caused the car not to steer hard enough in the corners. For the VGG type model I found that some dropout in the final layers improved performance.

## NVIDIA Type Model Structure and Parameters
| Layer | Size | Memory (Forward Pass) | # Parameters (Not Counting Bias) |
| ---- | :------------------:| --------:| ---------------:|
| input | 66 X 200 X 3 | 0.040 MB | 0 |
| conv1 | 31 X 98 X 24 | 0.073 MB | 1800 | 
| conv2 | 14 X 47 X 36 | 0.024 MB | 21600 | 
| conv3 | 5 X 22 X 48 | 0.005 MB | 43200 | 
| conv4 | 3 X 20 X 64 | 0.004 MB | 27648 | 
| conv5 | 1 X 18 X 64 | 0.001 MB | 36864 | 
| FC1 | 1 X 1 X 100 | 0.0001 MB | 115200 | 
| FC2 | 1 X 1 X 50 | 0.0001 MB | 5000 | 
| FC3 | 1 X 1 X 10 | 0.0001 MB | 500 | 

Based on the notes from Stanford's CS231n, this gives 0.6 MB for each image on forward pass and 1.2 MB on the backward pass. Using a batch size of 64, the max memory usage will be 75 MB during the backward pass.

## VGG Type Model Structure and Parameters
| Layer | Size | Memory (Forward Pass) | # Parameters (Not Counting Bias) |
| ---- | :------------------:| --------:| ---------------:|
| input | 64 X 64 X 3 | 0.012 MB | 0 |
| conv1 | 64 X 64 X 32 | 0.131 MB | 864 | 
| pool1 | 32 X 32 X 32 | 0.033 MB | 0 | 
| conv2 | 32 X 32 X 64 | 0.066 MB | 18432 | 
| pool2 | 16 X 16 X 64 | 0.016 MB | 0 | 
| conv3 | 16 X 16 X 128 | 0.033 MB | 73728 | 
| pool3 | 8 X 8 X 128 | 0.008 MB | 0 | 
| FC1 | 1 X 1 X 512 | 0.001 MB | 4194304 | 

This gives 1.2 MB (~0.3MB * 4 bytes) for each image on forward pass and 2.4 MB on the backward pass. Using a batch size of 64, the max memory usage will be 150 GB during the backward pass. Comparing the structure and parameters to NVIDIA’s model, at nearly 4.3 million parameters, this model has significantly more parameters than NVIDIA’s.

# Results
The following videos show side by side comparisons of the two models for both Track 1 and 2. While the VGG model appears to drive slightly smoother on Track 1, the NVIDIA model generalized better to Track 2. I was having trouble with overfitting using the VGG model and the results for the two tracks appear to support this finding.

## Driving on Track 1 with 0.2 Throttle Value

<img src="images/Track1_SBS.png">

Link to video: https://youtu.be/hUp1T83X8f4

## Driving on Track 2 with 0.3 Throttle Value

<img src="images/Track2_SBS.png">

Link to video: https://youtu.be/3O8e9H-RP5k

# Conclusion
Since I first saw the Udacity Nanodegree projects this was the one I was most looking forward to. With the difficulties associated with collecting high quality and quantity real world data on the road, being able to train these models in a simulator with seemingly infinite data could provide a significant breakthrough in autonomous driving. As seen in this project, even with a relatively small amount of data and fairly simply CNN model, the vehicle was able to steer successfully through both courses.

I look forward to continuing with with this work and have been working on implemententing an RNN/LSTM approach. I would also be curious to use combinations of real world and simulator data to train these model's.
