---
layout: post
title: 숫자 손글씨 이미지 딥러닝
date: 2022-12-24 03:08:45 +0900
category: ML
---

환경은 구글 Colab의 Jupyter Notebook이다. <br /> 
아래의 코드는 스파르타 코딩 클럽의 3주차 숙제를 바탕으로 구성되어진 것들이다.

```py
import os
from tensorflow.keras.models import Model
from tensorflow.keras.layers import Input, Dense
from tensorflow.keras.optimizers import Adam, SGD
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.preprocessing import OneHotEncoder

os.environ['KAGGLE_USERNAME'] = 'MY_USERNAME'
os.environ['KAGGLE_KEY'] = '###########MY_KEY###############'
```

<br />

MNIST의 데이터이다.

```bash
!kaggle datasets download -d oddrationale/mnist-in-csv
!unzip mnist-in-csv.zip
```

<br />

학습시킬 데이터 셋을 한번 확인해보고

```py
train_df = pd.read_csv('mnist_train.csv')

train_df.head()
```
<img src="../../../../img/ML/ML-handwritten-digits-001.png" width=800>

<br />

테스트 데이터 셋도 한번 보고

```py
test_df = pd.read_csv('mnist_test.csv')

test_df.head()
```
<img src="../../../../img/ML/ML-handwritten-digits-002.png" width=800>

<br />

라벨들은 보기 쉽게 그래프로 그려본 다음

```py
plt.figure(figsize=(16, 10))
sns.countplot(train_df['label'])
plt.show()
```

<img src="../../../../img/ML/ML-handwritten-digits-003.png" width=480>

<br />

입출력을 나누어 전처리를 실행한다.

```py
train_df = train_df.astype(np.float32)
x_train = train_df.drop(columns=['label'], axis=1).values
y_train = train_df[['label']].values

test_df = test_df.astype(np.float32)
x_test = test_df.drop(columns=['label'], axis=1).values
y_test = test_df[['label']].values

print(x_train.shape, y_train.shape)
print(x_test.shape, y_test.shape)
```

<img src="../../../../img/ML/ML-handwritten-digits-004.png" width=270>

<br />

한번 이미지 형태로 뽑아보자, 픽셀이 28x28이므로 reshape을 이용해 2차원 형태로 재구성해야 사람이 알아 볼 수 있을 것이다.

```py
index = 1
plt.title(str(y_train[index]))
plt.imshow(x_train[index].reshape((28, 28)), cmap='gray')
plt.show()
```

<img src="../../../../img/ML/ML-handwritten-digits-005.png" width=270>

<br />

이번엔 컴퓨터가 편하게 원핫 인코딩을 하자

```py
encoder = OneHotEncoder()
y_train = encoder.fit_transform(y_train).toarray()
y_test = encoder.fit_transform(y_test).toarray()

print(y_train.shape)
```

<img src="../../../../img/ML/ML-handwritten-digits-006.png" width=160>

<br />

각 자리에 어떤 라벨이 어느 순서로 들어가는지 확인할 수 있는 코드이다.
지금은 그래프 출력 때부터 보면 순서대로 되어있지만, 내 생각엔 다른 많은 종류의 데이터를 
다루게 된다면 순차적으로 된 숫자이라는 보장도 없고 데이터 안에서 라벨이 어느 순서로 되어있는지도 확인하려면 번거럽고 또 많으면 착오가 생길 수 있다는 생각이 들어 [참조 문서](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.OneHotEncoder.html)에 들어가 찾아낸 것이다. 이렇게 하면 몇 번 자리에 뭐가 해당하는지 정확히 쉽고 빠르게 알 수 있다.

```py
print(encoder.categories_)
```

<img src="../../../../img/ML/ML-handwritten-digits-007.png" width=700>

<br />

이미지 데이터는 256개의 픽셀 즉 0에서 255 사이 정수 값으로 되어 있으니 255로 나누어 0에서 1사이로 일반화를 시키도록 한다. 주의할 점은 실행 할 때마다 중복 적용되기에 절대로 두 번 이상 실행시키면 안된다는 점이다.

```py
x_train = x_train / 255.
x_test = x_test / 255.
```

<br />

이제 학습시킨다.
Input에는 28x28 이미지이니 784이고 출력은 0부터 9까지이니 Dense를 10개로 맞춰준다.
그 밖에 Dense를 어떻게 정하는게 가장 좋은 것인지 궁금하여 검색하여 본 결과, [StackOverFlow](https://stackoverflow.com/questions/36950394/how-to-decide-the-size-of-layers-in-keras-dense-method)에서의 중론을 참고하면 그리드 서치일 경우 2의 배수로, 랜덤 서치일 경우 무작위로 최적의 수를 여러번 시도해서 찾는 수 밖에 없는 것 같다. 이미지는 그리드 일테니 2의 배수 값들로 하도록 하자.
```py
input = Input(shape=(784,))
hidden = Dense(512, activation='relu')(input)
hidden = Dense(1024, activation='relu')(hidden)
hidden = Dense(256, activation='relu')(hidden)
output = Dense(10, activation='softmax')(hidden)

model = Model(inputs=input, outputs=output)

model.compile(loss='categorical_crossentropy', optimizer=SGD(lr=0.001), metrics=['acc'])

model.summary()
```

<img src="../../../../img/ML/ML-handwritten-digits-008.png" width=600>

<br />

그대로 학습시킨다.

```py
history = model.fit(
    x_train,
    y_train,
    validation_data=(x_test, y_test),
    epochs=20
)
```

<img src="../../../../img/ML/ML-handwritten-digits-009.png" width=1000>

나는 이번엔 SGD를 의도적으로 사용했다. Adam으로 Dense값을 조절해서 과적합도 잡아보고 했지만 오차가 의미 있는 수준으로 줄지도 않을 뿐더러 테스트 값에 비슷하게 도달할 수 조차 없어서 어디를 조절해야 할지 고민하다가 저번에 검색할 때 Adam은 이미지 처리에 있어선 SGD보다 좋지 않다는 평가가 있다는 말이 생각이 나서 바꾸어 봤더니 오히려 이번 건에서 만큼은 아주 미약하게나마 테스트 정확도가 학습 정확도보다 높게 나올정도로 잭팟이다.

<br />

그래프로 그 차이를 실감해 보자

```py
plt.figure(figsize=(16, 10))
plt.plot(history.history['loss'])
plt.plot(history.history['val_loss'])
```
<figure>
    <img src="../../../../img/ML/ML-handwritten-digits-010.png" width=800>
    <figcaption>SGD</figcaption>
</figure>

<figure>
    <img src="../../../../img/ML/ML-handwritten-digits-012.png" width=800>
    <figcaption>Adam</figcaption>
</figure>

```py
plt.figure(figsize=(16, 10))
plt.plot(history.history['acc'])
plt.plot(history.history['val_acc'])
```

<figure>
    <img src="../../../../img/ML/ML-handwritten-digits-011.png" width=800>
    <figcaption>SGD</figcaption>
</figure>

<figure>
    <img src="../../../../img/ML/ML-handwritten-digits-013.png" width=800>
    <figcaption>Adam</figcaption>
</figure>

<br />

Adam인 경우의 Dense 값은 조금 다르긴하다 SGD를 사용할 때 쯤 몇 번 고쳐서 그렇긴한데, 고쳐서 매번 실험했던게 Adam이었고 계속 지켜본 결과로는 뭐... 여기선 별반 의미있게 다르지 않기에 그냥 위의 스크린샷을 첨부했다. 아무튼 계속 하면서 느끼는 건, 재밌다 이거.