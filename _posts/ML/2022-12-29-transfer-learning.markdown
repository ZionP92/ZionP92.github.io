---
layout: post
title: 전이 학습
date: 2022-12-29 13:43:34 +0900
category: ML
---

환경은 구글 Colab의 Jupyter Notebook이다. <br /> 
아래의 코드는 스파르타 코딩 클럽의 4주차 숙제를 바탕으로 구성되어진 것들이다.

```py
import os
from keras.models import Model
from keras.layers import Input, Dense, Conv2D, MaxPooling2D, Flatten, Dropout
from keras.optimizers import Adam, SGD
from keras.preprocessing.image import ImageDataGenerator
from keras.applications import ResNet50
from keras.callbacks import ModelCheckpoint
from keras.models import load_model
import keras.utils as image
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.preprocessing import OneHotEncoder
from pprint import pprint

os.environ['KAGGLE_USERNAME'] = 'MY_USERNAME'
os.environ['KAGGLE_KEY'] = '###########MY_KEY###############'
```

<br />

분류할 이미지 데이터이다. unzip의 -q는 quiet로 이번 데이터가 많아서 너무 다량의 출력 결과가 나오니 출력하지 못하게 제한한다.
```bash
!kaggle datasets download -d puneet6060/intel-image-classification
!unzip -q intel-image-classification.zip
```

<br />

ImageDataGenerator로 이미지 데이터 증강을 하는데 rescale은 보다시피 일반화, rotation_range는 랜덤하게 이미지를 회전할 각도 0-180중 범위의 최대값을 기입하고 zoom_range, width_shift_range, height_shift_range 즉, 확대 수평 수동이동에도 마찬가지로 범위 최대의 퍼센트 값을 준다. horizontal_flip 역시 말 그대로 랜덤으로 수평 뒤집기이다.

flow_from_directory를 사용하면 하위 폴더를 기준으로 클래스 별로 알아서 분리해준다.
이진 분리일 경우에 categorical을 binary로 바꾸어주면 됨.

여기의 seed는 train_test_split의 random_state와 같은 기능을 한다.

한 가지 앞선 예시들에서 target_size는 왜 15의 제곱으로 하던데 왜 그렇게 정했는지는 잘 모르겠다. 이미지들을 다 해당 사이즈로 바꿔주는 기능인데 기본값은 16의 제곱이고 난 그냥 이미지를 확인해보니 파일 사이즈가 150x150이라 더 줄이지는 않고 그대로 넣었다.(구글링 했을 때 많이 예시로 나온 사이즈가 150x150이기도 했으니. 예시에서 있던 파일 사이즈들은 내가 해본 시점에서는 변경되었을지도 모르지만 100x100이었는데 없는 데이터를 늘려도 잡음일텐데 정확히 무엇을 기준으로 하는 건지, 지금까지 알아본 바로는 너무 손상 안될 정도로 사이즈를 줄여 학습에 용이하게(?) 한다 정도인데 아무튼 계속 연구해 봐야겠다.)

테스트 부분은 결과를 위해 셔플하지 않는다.
```py
train_datagen = ImageDataGenerator(
  rescale=1./255,
  rotation_range=10,
  zoom_range=0.1,
  width_shift_range=0.1,
  height_shift_range=0.1,
  horizontal_flip=True
)

test_datagen = ImageDataGenerator(
  rescale=1./255
)

train_gen = train_datagen.flow_from_directory(
  'seg_train/seg_train',
  target_size=(150, 150),
  batch_size=32,
  seed=2021,
  class_mode='categorical',
  shuffle=True
)

test_gen = test_datagen.flow_from_directory(
  'seg_test/seg_test',
  target_size=(150, 150),
  batch_size=32,
  seed=2021,
  class_mode='categorical',
  shuffle=False
)
```
<img src="../../../../img/ML/ML-transfer-learning-001.png" width=350>

<br />

하위 폴더 여섯개가 각각 클래스로 적립됐다. 아래 코드를 이용해 어떻게 엮였는지 본다.
```py
pprint(train_gen.class_indices)
```

<img src="../../../../img/ML/ML-transfer-learning-002.png" width=150>

<br />

후에 로딩만해서 쓰면 분류명을 알 수가 없다. 다른 방법은 모르겠고 난 따로 저장을 했다.
```py
np.save('class_indices', train_gen.class_indices)
```

<br />

증강되어 있어서 이미지가 올곧지 않을 수 있다. 그래도 한번 봐보자.
```py
preview_batch = train_gen.__getitem__(0)

preview_imgs, preview_labels = preview_batch

plt.title(str(preview_labels[0]))
plt.imshow(preview_imgs[0])
```

<img src="../../../../img/ML/ML-transfer-learning-003.png" width=250>

<br />

전이 학습 시킨다. 불러온 모델은 ResNet50이다.
imagenet을 학습시키는 weight를 사용하는건데 이니셜 웨이트에 불과하다. 곧 유리한 위치에서 시작해서 사용자가 넣는 데이터에 맞춰 계속 변환된다. include_top은 이미 정해진 출력레이어를 사용할 것인지에 관한 설정인데 출력레이어를 마음대로 바꿔야하니 False로 놓는다.
x를 출력으로 정의하고 Dropout 및 Dense를 지정하는데 출력 Dense는 클래스 갯수에 맞춰 6개로 지정하고 서머리를 뽑아보면 마지막 Dropout부터만 사용자 정의이고 나머지는 ResNet50 모델 값이 출력이 된다. 너무 기니 스크린샷은 아랫부분만 담았다.
```py
input = Input(shape=(150, 150, 3))

base_model = ResNet50(weights='imagenet', include_top=False, input_tensor=input, pooling='max')

x = base_model.output
x = Dropout(rate=0.25)(x)
x = Dense(256, activation='relu')(x)
output = Dense(6, activation='softmax')(x)

model = Model(inputs=base_model.input, outputs=output)

model.compile(loss='categorical_crossentropy', optimizer=SGD(learning_rate=0.001), metrics=['acc'])

model.summary()
```

<img src="../../../../img/ML/ML-transfer-learning-004.png" width=670>

학습을 시키고 ModelCheckpoint를 이용하여 학습 결과 저장을 하는데 .h5 형식의 파일로 저장을 한다. 파라미터는 val_acc 값을 주시하고 최고값만 저장하게 설정한다. verbose=1은 저장시 저장했음을 알려달라는 얘기
```py
history = model.fit(
    train_gen,
    validation_data=test_gen,
    epochs=10,
    callbacks=[
      ModelCheckpoint('model.h5', monitor='val_acc', verbose=1, save_best_only=True)
    ]
)
```

<img src="../../../../img/ML/ML-transfer-learning-005.png" width=890>

<br />

그래프로 출력해보자
```py
fig, axes = plt.subplots(1, 2, figsize=(20, 6))
axes[0].plot(history.history['loss'], label="loss")
axes[0].plot(history.history['val_loss'], label="val_loss")
axes[1].plot(history.history['acc'], label="acc")
axes[1].plot(history.history['val_acc'], label="val_acc")
axes[0].legend()
axes[1].legend()
```

<img src="../../../../img/ML/ML-transfer-learning-006.png" width=1200>

<br />

테스트 데이터를 분류해본다.
5번째 데이터를 넣은 것이고 classes에 인덱스로 접근하는데 one-hot encoding된 부분 매칭 값이 예측값으로 나오니 그 중 최대 예측값의 인덱스 번호를 뽑아내야 한다.
학습은 그럭저럭 잘 된듯 싶다.
```py
test_imgs, test_labels = test_gen.__getitem__(5)

y_pred = model.predict(test_imgs)

classes = dict((v, k) for k, v in test_gen.class_indices.items())

fig, axes = plt.subplots(4, 8, figsize=(20, 12))

for img, test_label, pred_label, ax in zip(test_imgs, test_labels, y_pred, axes.flatten()):
  test_label = classes[np.argmax(test_label)]
  pred_label = classes[np.argmax(pred_label)]

  ax.set_title('GT:%s\nPR:%s' % (test_label, pred_label))
  ax.imshow(img)
```

<img src="../../../../img/ML/ML-transfer-learning-007.png" width=1500>

<br />

다음으로는 저장된 학습시킨 모델을 불러오는 법이다.

```py
model = load_model('model.h5')

print('Model loaded!')
```

<img src="../../../../img/ML/ML-transfer-learning-008.png" width=120>

<br />

여기서 부터는 예시가 없었어서 그냥 구글링과 내 생각대로 코드를 짜집기했다.
(정상 작동하나 비효율적일지도?)
이미지들을 불러와서 읽을 수 있게 처리해주자
```py
target_imgs = []
target_path = "seg_pred/seg_pred"

for img in os.listdir(target_path):
    img = os.path.join(target_path, img)
    img = image.load_img(img, target_size=(150, 150))
    img = image.img_to_array(img)
    img = np.expand_dims(img, axis=0)
    processed_img = np.array(img, dtype="float")/255.0
    target_imgs.append(processed_img)
  
target_imgs = np.vstack(target_imgs)
```

<br />

이미지 불러오는 부분과 아래를 나눈 이유는 혹시라도 에러가 나면 불러오는 과정을 계속 하기에 비효율 적이라 나누었다. 해보니 적극 추천

아래는 거의 비슷한데 아까 저장한 .npy 파일을 불러와서 읽어줘야 한다.
그냥 만들면서 쭉하면 응당 쉽게 위의 코드로 라벨 정답값 부분만 해결하면 될텐데 핵심만 뽑아쓰면 클래스 부분이 비어 안될 것 같아서 해보니 정말 그러하였다. 그래서 이것도 나중에 내 편의를 위해 구성해 보았다.
```py
predicted = model.predict(target_imgs)

fig, axes = plt.subplots(4, 8, figsize=(20, 12))

classes = dict((v, k) for k, v in np.load('class_indices.npy', allow_pickle=True).item().items())

for img, pred_label, ax in zip(target_imgs, predicted, axes.flatten()):
  pred_label = classes[np.argmax(pred_label)]

  ax.set_title('PR:%s' % pred_label)
  ax.imshow(img)
```

<img src="../../../../img/ML/ML-transfer-learning-009.png" width=1500>

<br />

예전부터 쭉 궁금했던 저장 불러오기와 전이 학습 등을 배울 수 있어서 정말 알찼던 거 같다. 예전에 이런 저런 코딩 배울 때처럼 모든 부분 혼자서 끙끙되며 해볼 수도 있었겠지만 그래도 어느정도 기초는 쉽게 가도 나쁘지 않은 것 같다.