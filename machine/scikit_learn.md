## 写在前面的话
如果用电脑浏览器打开GitHub的.md文件，有些公式显示不出来，可以参考这篇资料[显示github中markdown文档的公式](https://blog.csdn.net/bitcarmanlee/article/details/87829033)



## scikit-learn源码分析
整个SKlearn项目，核心的部分在sklearn这个包里面，算法的使用案例在example包里面

![scikit-learn 项目结构](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200306162555246-556833908.png)


## sklearn包的简介
* _check_build
简单的检查是否正确编译的脚本
* _build_utils
sklearn官方团队构建这个项目时使用的一些支持性工具
* _loss
GLM算法中会用到的分布函数
* cluster
聚类算法的实现包。含有Birch/DBSCAN/Hierarchical/KMeans/Spectral等算法，其中kmeans算法提供elkan/fast/lloyd几种实现方式(使用cpyton实现的)。
* compose
合成模型时使用的元学习器。
* covariance
计算特征间协方差
* cross_decomposition
包含CCA（典型相关分析）和PLS（偏最小二乘）两种算法，这些算法主要用于探索两个多元数据集之间的线型关系。
* datasets
sklearn自带的数据采集器，主要功能时响应用户的调用，从网络下载玩具数据集，方便用户简单地跑跑算法。
* decompostion
矩阵分解算法包。包括PLA（主成分分析）、NMF（非负矩阵分解）、ICA（独立成分分析），其中PLA有稀疏版本的实现。
* ensemble
集成算法包。主要的脚本有：_bagging.py(bagging算法),_forest.py(随机森林),_gb(GBDT),_iforest.py(孤独森林)，_stacking.py（stacking方法）,_voting.py(投票方法),_weight_boosting.py(主要就是AdaBoost算法)。注意，GBDT算法的实现过程有调用一个cpython版本的脚本_gradient_boosting.pyx，主要目的就是加快计算速度。
* experimental
实验模块
* externals
一些外部依赖脚本。
* feature_extraction
特征提取。目前支持同Text文档和图片中提取特征。
* feature_selection
特征选择算法，主要是单变量的过滤算法（例如去除方差很小的特征）和递归特征删除算法。这里的接口也可以用来做降维。
* guassian_process
高斯过程。分为分类和回归两个实现。
* impute
缺失值处理，例如使用KNN算法去填充缺失值。
* inspection
检查模型的工具
* linear_model
线性模型包，包含线性回归、逻辑回归、LASSO、Ridge、感知机等等，内容非常多，是sklearn中的重点包
* manifold
实现数据嵌入技术的工具包，包括LLE、Iosmap、TSNE等等
* metrics
集合了所有的度量工具，包括常见的accuracy_socre、auc、f1_score、hinge_loss、roc_auc_score、roc_curve等等
* mixture
高斯混合模型、贝叶斯混合模型
* model_selection
sklearn的重点训练工具包，包括常见的GridSearchCV、TimeSeriesSplit、KFold、cross_validate等
* neighbors
K近邻算法包，包括球树、KD树、KNN分类、KNN回归等算法的实现
* neutral_network
包含比较基础的神经网络模型，例如伯努利受限玻尔兹曼机、多层感知机分类、多层感知机回归
* preprocessing
sklearn的重点数据预处理工具包，包括常见的LabelEncoder、MinMaxScaler、Normalizer、OneHotEncoder等
* semi_supervises
半监督学习算法包，LabelPropagation、LabelSpreading
* svm
支持向量机算法包，包括线性支持向量分类/回归、SVC/SVR、OneClassSVM等，但是sklearn自己并没有独立去实现这一类算法，而是复用了很多libSVM的代码
* tests
一些单元测试代码
* tree
树模型，包括决策树、极端树。注意，梯度提升树算法放在ensemble包里。
* utils
加速计算、cython版本的BLAS算法、优化等工具包

sklearn项目可以看成一棵大树，各种estimator是果实，而支撑这些估计器的主干，是为数不多的几个基类。常见的几个类有BaseEstimator、BaseSGD、ClassifierMixin、RegressorMixin，等等
官方文档的API参考页面列出了主要的API接口，我们看下Base类

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320212850771-1859142633.png)

## BaseEstimator
最底层的就是BaseEstimator类。主要暴露两个方法：set_params，get_params

### get_params函数细节
这个方法旨在获取对象的参数，返回对象默认是{参数:参数值}的键值对。如果将get_params的参数deep设置为True，还会返回（如果有的话）子对象（它们是估计器）。下面我们来仔细看一下这个方法的实现细节

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320165119520-70294856.png)

（1）函数体中主要就是**getattr**方法，语法：getattr(对象，要检索的属性[，如果属性不存在则返回的值])。Line200~208的任务是判断self（一般就是估计器的实例）是否含有key这个参数，如果有就返回它的参数值，否则人为设置为None

（2）再来看Line209~212，如果用户设置了**deep=True**，并且value对象实现了get_params（说明value对象是一个子对象，即估计器，否则普通的参数是不会再次实现get_params方法的），则提取参数字典的键值对，并且写入字典。整个函数最后返回的也是字典。
（3）快速的看一下这个方法具体是怎么使用的，然后再继续追踪源码的实现。

```python
from sklearn.ensemble import RandomForestClassifier
clf = RandomForestClassifier(random_state=0)
X = [[ 1,  2,  3],  # 2 samples, 3 features
     [11, 12, 13]]
y = [0, 1]  # classes of each sample
clf.fit(X, y)
```


简单的实例化一个随机森林分类器的对象，我们看下对它调用**get_params**会返回什么：

```bash
clf.get_params()
{'bootstrap': True,
 'class_weight': None,
 'criterion': 'gini',
 'max_depth': None,
 'max_features': 'auto',
 'max_leaf_nodes': None,
 'min_impurity_decrease': 0.0,
 'min_impurity_split': None,
 'min_samples_leaf': 1,
 'min_samples_split': 2,
 'min_weight_fraction_leaf': 0.0,
 'n_estimators': 10,
 'n_jobs': None,
 'oob_score': False,
 'random_state': 0,
 'verbose': 0,
 'warm_start': False}
```

很明显，这就是这个随机森林分类器的默认参数方案。
（4）我们注意到Line199这行，使用了另一个方法**for key in self._get_param_names():**，现在研究该函数

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320165025000-1703711635.png)


**@classmethod**这个装饰器直接告诉我们，该方法的适用对象是类自身，而非实例对象。
这个函数有很多检查事项，真正获取参数的是**inspect.signature(init).parameters.values()**，最后获取列表中每个对象的name属性。

## set_param_names
这个方法作用是设置参数。正常来说，我们在初始化估计器的时候定制化参数，但是也有临时修改参数的需求，这时可以手工调用**set_params**方法。但是更多的还是由继承BaseEstimator的类来调用这个方法。

具体地，我们看下实现细节：

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320164932289-1270693722.png)

这个方案支持处理嵌套字典，但是我们不去纠缠这么琐碎，直接看到L251，**setattr(self, key, value)**，对估计器的key属性设置一个新的值。

应用的实例：

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320170223749-1093534343.png)

## ClassifierMixin
Mixin表示混入类，可以简单地理解为给其他的类增加一些额外的方法。Sklearn的分类、回归混入类只实现了**score**方法，任何继承它们的类需要自己去实现**fit、predict**等其他方法。

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320220339008-43416124.png)

关于混入类，简单的说就是一个父类，但是和普通的类有点不同，它需要指明元对象，**_estimator_type**。这里不再展开论述，感兴趣的读者请阅读这篇讨论[What is a mixin, and why are they useful?](https://stackoverflow.com/questions/533631/what-is-a-mixin-and-why-are-they-useful)

可以看到，这个混入类的实现非常简单，求预测值和真实值的准确率，返回值是一个浮点数。注意预测值来自**self.predict()**，所以继承混入类的类必须自己实现**predict**方法，否则引发错误。后面不再重复强调该细节。

再次的，分类任务的混入类又是在搬运其它函数的劳动成果，那我们就来研究一下**accuracy_score**的实现细节

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320221717789-1246600479.png)

为简洁起见，我们先忽略L185~189之间的代码，后面会有专门研究分类任务的度量方法的文章，在那里我们再仔细研究它。直接看L191，y_ture == y_pred，这是一个简单的写法，精妙在于避免了for循环，快速的检查两个对象之间每一个元素是否相等并且返回True/False。L193对score结果做一层包装。

* L116：如果设置了normalize参数为True，则对score列表取平均值，就是预测正确的样本个数/总体个数=预测准确率
* L118：如果有权重，则按照权重对各个样本的得分进行加权，作为最终的预测准确率
* L121：如果没有上述两种设置，则直接返回预测正确的样本的个数。注意：sklearn默认的score方法返回预测准确率，而非预测正确的样本个数。

## RegressorMixin
![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200320223428058-1221630186.png)

毫不意外地，回归任务的混入类只实现了**score**方法，核心数学原理是$R^2$值。公式是$1-\frac{({y_true} - {y_pred})^2}{({y_true} - {y_true_mean})^2}$，直观上看，这个值是衡量预测值与真实值的偏离度与真实值自身偏离度的一个比值。 R2最大为1，表示预测完全准确，值为0时表示模型没有任何预测能力。

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323100147939-433562934.png)

**score**方法调用了**metrics**模块的**r2_score**方法，返回值是浮点数。我们来研究下**r2_score**，这个函数是目前为止我们看过的最复杂的一个。因此，我们一块一块来研究。

## 检查传入的对象
![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323101338357-550997761.png)

（1）检查传入对象的长度
L577调用**check_consistent_length**检查输入标签、输出标签、权重是不是有相同的长度。检查的方法也很简单，对每个对象计算长度，然后取不同的长度值有多少个，如果超过1个，说明几个对象之间的长度不一，则引发一个错误来警告。

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323101951776-9847086.png)

（2）检查传入的参数是否合法
L575调用**_check_reg_targets**方法，旨在检查传入参数是否合法。

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323102402643-1848500840.png)


这个函数略长，但是大致做了以下几件事：

* L83~95都是在做检查和格式转换。
* L97~114检查输入multioutput和y_true是否吻合，即真实的标签数组的维度如果是1的话，显然设置multioutput这个参数非None是不合法的。并且当真实标签数组的维度大于1的时候，若其维度和multioutput不同时也会引发错误以告警。
* L115根据y_true的维度决定标签是哪种类型，分为：连续型和多类输出的连续型。

```diff
- 注意：multioutput可以是字符串，也可以是一个数组，还可以是None值（考虑到向下兼容），因此这个参数非常灵活。后面研究具体算法时遇到了会再次提及，此处不作过多纠缠。
```

## 检查样本数和权重系数
继续看r2_score的实现：

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323103834041-709589152.png)

（3）L597~582检查预测值的样本数
如果预测值的样本数不足2个，则引发错误告警。因为决定系数(即$R^2$)要求至少要有2个样本

（4）L584~588处理权重系数
* L585调用np.ravel()，把权重数组拉平到一维
* L586对sample_weights扩维，将一维扩充为二维，二维扩充为三维，以此类推。值得注意的是，np.newaxis放置的位置不同，扩充的方向是不同的，具体看下面这个小例子：

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323104932112-1784236917.png)

* L588，如果没有传入权重系数，则默认设置为1

## 实现$R^2$的计算细节
（5）构造分子和分母

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323105501330-2135896065.png)

（6）计算每个样本的得分

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323105719323-433945894.png)

* L595~596 记录分母和分子的数组中不为0的索引值(就是非0值所在的位置)
* L597 记录分子、分母同时不为0的样本的索引值。如果对这个写法不熟悉，这里有个小例子帮助理解：

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323105950880-1851528546.png)

* L598~599 创建一个和真实标签相同长度的全1数组，然后对合法的索引位置计算真实的$R^2$值。
* L603 将分母为0的索引位置的值设置为0，这里设为其他常数也是可以的，对于同一个回归任务的评价没有影响。

（7）根据multioutput参数来决定各样本所得分数的权重

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323110439427-347254051.png)

* L605~607 如果指明raw_values，则输出每个样本的分数
* L608~610 如果指明uniform_average，则avg_weights设置为None，其实就是均匀分布权重
* L611~612 如果指明variance_weighted，则直接用分母作权重
* L614~618 处理常量y值或一维数组的情形。如果分母全是0，则：若分子有非0，直接返回1；否则返回0
* L620 如果multioutput不是字符串，则直接把它作为最后的权重系数

（8）返回得分

```bash
return np.average(output_scores, weights=avg_weights)
```

刚刚说到，指明uniform_average，则avg_weights设置为None。在numpy.average这个方法里，如果权重是None，计算均值就是简单的mean()函数。

## TransformerMixin

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323112754626-781647688.png)

这个混入类的实现比较简单，完全依靠使用它的类自己实现的fit方法和transform方法。但是它会根据是否有标签，决定是有监督任务还是无监督任务。等后面遇到再具体讨论。

![](https://img2020.cnblogs.com/blog/1342077/202003/1342077-20200323113102323-948276536.png)



1.[简单实现的scikit-learn neighbors][2]

2.[sklearn 源码分析系列：neighbors][3]


[2]:https://github.com/demonSong/DML
[3]:https://blog.csdn.net/u014688145/article/details/61916582
