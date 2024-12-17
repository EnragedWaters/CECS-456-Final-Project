/*
BY TROY WATERS
STUDENT ID: 029451303

Both the Google collab and Python file are included based off preference.
IT SHOULD BE NOTED I STORED A LOCAL VERSION OF THE 26,000 IMAGES ONTO MY GOOGLE DRIVE FOR COMFORT,
sadly the main website from which the dataset is derived from would randomly disconnect, and proved to
be inconsistent. 

A raw copy-paste of the code will go here as well:
*/

# By Troy Waters (solo, no team members)
# CECS 456 Final Project
# Student ID : 029451303

import os
import numpy as np
from tensorflow.keras.preprocessing.image import ImageDataGenerator
from tensorflow.keras.models import Sequential, load_model
from tensorflow.keras.layers import Conv2D, MaxPooling2D, Flatten, Dense, Dropout
from tensorflow.keras.callbacks import ModelCheckpoint
import matplotlib.pyplot as plt

# All ~26,000 images are stored on my personal Google Drive
# This is due to issues with obtaining the images directly 
# from the website, and this was the only consistent 
# way to get results on my low-end laptop
dataset_path = '/content/drive/My Drive/archive/raw-img'
from google.colab import drive
drive.mount('/content/drive')

# datagen will handle normalizing pixels
# because PC's handle decimate calculations (0 to 1)
# better than large integers ( such as 0 to 255)
datagen = ImageDataGenerator(
    rescale=1./255,  # normalize
    validation_split=0.3  # Splitting for validation later
)

# Given Google collab RAM constraints
# Images are resized to half their original size
# And batches of 32 images are handled at a time
# To avoid early termination
train_generator = datagen.flow_from_directory(
    dataset_path,
    target_size=(128, 128),  # 255x255 px to 128x128 px
    batch_size=32, #32 images at a time 
    class_mode='categorical', # 10 classes of animals
    subset='training' 
)

# validation set has identical
# properties to the training set
# to maintain consistency
val_generator = datagen.flow_from_directory(
    dataset_path,
    target_size=(128, 128),
    batch_size=32,
    class_mode='categorical',
    subset='validation' 
)

# Creation of the Sequential CNN model

def create_model(input_shape, num_classes):
    model = Sequential([
        Conv2D(32, (3, 3), activation='relu', input_shape=input_shape),
        MaxPooling2D((2, 2)), # Shrinking feature map for optimization
        Conv2D(64, (3, 3), activation='relu'),
        MaxPooling2D((2, 2)), # Shrinking once more
        Flatten(), # Turning 2D feature maps to 1D
        Dense(128, activation='relu'),
        Dropout(0.5), # To avoid overfittin
        Dense(num_classes, activation='softmax')
    ])
    model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy'])
    return model

# 128x128 px, and 3 color channels in the image (Red, Green, Blue)
input_shape = (128, 128, 3)
num_classes = len(train_generator.class_indices)

# Since google collab can terminate when you least expect it
# 'check points' were added
# as a safety measure, so we don't need to re-run 10 hours of lost time
if os.path.exists('best_animal_classifier.keras'):
    print("Loading saved model...")
    model = load_model('best_animal_classifier.keras')
else:
    print("Creating a new model...")
    model = create_model(input_shape, num_classes)

# saving every epoch for sanity sake...
checkpoint = ModelCheckpoint(
    'best_animal_classifier.keras',
    monitor='val_accuracy',
    save_best_only=True,
    verbose=1
)

# Start at epoch 0, but check if we have previous model
initial_epoch = 0
if os.path.exists('training_history.log'):
    with open('training_history.log', 'r') as f:
        lines = f.readlines()
        if lines:
            initial_epoch = int(lines[-1].strip()) + 1

history = model.fit(
    train_generator,
    epochs=20, 
    validation_data=val_generator,
    initial_epoch=initial_epoch,  # checkpoint would go here if it exists
    callbacks=[checkpoint]
)

# Saving training
with open('training_history.log', 'a') as f:
    for epoch in range(initial_epoch, 20):
        f.write(f"{epoch}\n")

# Evaluating the validation models for highest accuracy
test_model = load_model('best_animal_classifier.keras')
test_loss, test_acc = test_model.evaluate(val_generator)
print(f"Test Accuracy: {test_acc:.2f}")

# visualization of all the data once complete
plt.plot(history.history['accuracy'], label='Training Accuracy')
plt.plot(history.history['val_accuracy'], label='Validation Accuracy')
plt.legend()
plt.xlabel('Epochs')
plt.ylabel('Accuracy')
plt.title('Training and Validation Accuracy')
plt.show()

plt.plot(history.history['loss'], label='Training Loss')
plt.plot(history.history['val_loss'], label='Validation Loss')
plt.legend()
plt.xlabel('Epochs')
plt.ylabel('Loss')
plt.title('Training and Validation Loss')
plt.show()