# 创建你的模型
sklearn 模型文件格式是 joblib， 参考如下的例子创建自己的模型
## 例子：鸢尾花（Iris）多分类
iris的分类是一个典型的人工智能分类问题，选取的是比较典型特点的三种鸢尾花：山鸢尾Iris setosa(0)、变色鸢尾Iris versicolor (1)、维吉尼亚鸢尾Iris virginica (2)，通过花的四个特征确定了单株鸢尾花的种类，这个四个特征是鸢尾花花瓣（petals）的长度和宽度、花萼（sepals）的长度和宽度，关于该例子的其他详细资料请参考：[ 鸢尾花（Iris）](https://blog.csdn.net/heivy/article/details/100512264)

本例子模型通过输入4个特征值，最终推测花的种类
## Creating your own model and testing the SKLearn Server locally.

To test the [Scikit-Learn](https://scikit-learn.org/stable/) server, first we need to generate a simple scikit-learn model using Python. 

```python 3.7
from sklearn import svm
from sklearn import datasets
from joblib import dump
clf = svm.SVC(gamma='scale')
iris = datasets.load_iris()
X, y = iris.data, iris.target
clf.fit(X, y)
dump(clf, 'model.joblib')
```
* 注意，本地测试的时候，本地安装的Scikit-learn的版本最好和AIMP的OOB的模型服务器中的sikit-learn的版本一致，即0.20.3版本，这样能够保证本地测试的效果和在kfserving中运行的效果保持一致。
* [sikit learn 官方安装文档](https://scikit-learn.org/stable/install.html)

Then, we can install and run the [SKLearn Server](../../../../../python/sklearnserver) using the generated model and test for prediction. Models can be on local filesystem, S3 compatible object storage, Azure Blob Storage, or Google Cloud Storage.

```shell
# we should indicate the directory containing the model file (model.joblib) by --model_dir
python -m sklearnserver --model_dir ./  --model_name svm
```

We can also use the inbuilt sklearn support for sample datasets and do some simple predictions

```python
from sklearn import datasets
import requests
iris = datasets.load_iris()
X, y = iris.data, iris.target
formData = {
    'instances': X[0:1].tolist()
}
res = requests.post('http://localhost:8080/v1/models/svm:predict', json=formData)
print(res)
print(res.text)
```
# 使用模型推理服务

## Setup
1. 中台运行正常
## Create the 推理服务

在中台模型UI中粘贴进去 sklearn.yaml文件的内容
![中台模型UI](/docs/diagrams/createrInferUI.png)

或者Apply the CRD
```
kubectl apply -f sklearn.yaml
```

Expected Output
```
$ inferenceservice.serving.kubeflow.org/sklearn-iris created
```
## Run a prediction （不需要access token）

```
MODEL_NAME=iris
INPUT_PATH=@./iris-input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice iris -o jsonpath='{.status.url}' -n mp | cut -d "/" -f 3)
INGRESS_HOST=$SERVICE_HOSTNAME
INGRESS_PORT=80
curl -v -H "Host: ${SERVICE_HOSTNAME}" http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict -d $INPUT_PATH
echo ""
```

Expected Output

```
*   Trying 169.63.251.68...
* TCP_NODELAY set
* Connected to 169.63.251.68 (169.63.251.68) port 80 (#0)
> POST /models/sklearn-iris:predict HTTP/1.1
> Host: sklearn-iris.default.svc.cluster.local
> User-Agent: curl/7.60.0
> Accept: */*
> Content-Length: 76
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 76 out of 76 bytes
< HTTP/1.1 200 OK
< content-length: 23
< content-type: application/json; charset=UTF-8
< date: Mon, 20 May 2019 20:49:02 GMT
< server: istio-envoy
< x-envoy-upstream-service-time: 1943
<
* Connection #0 to host 169.63.251.68 left intact
{"predictions": [1, 1]}
# 如果出现如下权限问题，则该推理服务需要access token
{"code":16, "message":"Missing or invalid \"authorization\" header.", "details":[]}
```
## Run a prediction （需要access token）
请参考 [《模型推理服务使用方式》章节](/README.md)
## （参考）Run SKLearn InferenceService with your own image
Since the KFServing SKLearnServer image is built from a specific version of `scikit-learn` pip package, sometimes it might not be compatible with the pickled model
you saved from your training environment, however you can build your own SKLearnServer image following [these instructions](../../../../../python/sklearnserver/README.md#building-your-own-scikit-learn-server-docker-image
).

To use your SKLearnServer image:
- Add the image to the KFServing [configmap](../../../config/configmap/inferenceservice.yaml)
```yaml
        "sklearn": {
            "image": "<your-dockerhub-id>/kfserving/sklearnserver",
        },
```
- Specify the `runtimeVersion` on `InferenceService` spec
```yaml
apiVersion: "serving.kubeflow.org/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    sklearn:
      storageUri: "gs://kfserving-samples/models/sklearn/iris"
      runtimeVersion: X.X.X
```
