# Validation schemes for 2-nd level models

Reivew Week1 2019-03-24

## *f) KFold scheme in time series*

In time-series task we usually have a fixed period of time we are asked to predict. Like day, week, month or arbitrary period with duration of T.

Split the train data into chunks of duration T. Select first M chunks.
Fit N diverse models on those M chunks and predict for the chunk M+1. Then fit those models on first M+1 chunks and predict for chunk M+2 and so on, until you hit the end. After that use all train data to fit models and get predictions for test. Now we will have meta-features for the chunks starting from number M+1 as well as meta-features for the test.
Now we can use meta-features from first K chunks [M+1,M+2,..,M+K] to fit level 2 models and validate them on chunk M+K+1. Essentially we are back to step 1. with the lesser amount of chunks and meta-features instead of features.


**f) KFold 方案在时间相关问题的应用**

在时间序列相关任务中我们通常是在一个固定的时间段上进行预测。如天，周，月或者持续时间为 T 的任意时间段。

1. 将训练数据按照持续时间 T 分割为若干块。选择前 M 个块。
2. 用 N 个不同的模型在这 M 个块上进行训练并且对 M+1 块进行预测。然后用这些模型在前 M+1 块上训练，再对 M+2 块进行预测以此类推，直到数据结束。在这之后，用全部训练数据训练这些模型并得到测试集的预测结果。现在我们得到了 从 M+1 块开始的 meta-features 以及 测试集的 meta-feature
3. 现在我们能够用前 K 个块即 [M+1,M+2,..,M+K] 个 meta-features 去训练第二层的模型并且在 M+K+1 块上进行验证。实质上我们返回到步骤 1 用较少的块和 meta-features 来代替 features。

```python
# 从dates_train 中取 28，29，30，31，32, 33 的行数据
dates_train_level2 = dates_train[dates_train.isin([27, 28, 29, 30, 31, 32, 33])] 
 
# That is how we get target for the 2nd level dataset
y_train_level2 = y_train[dates_train.isin([27, 28, 29, 30, 31, 32, 33])]

# And here we create 2nd level feeature matrix, init it with zeros first
X_train_level2 = np.zeros([y_train_level2.shape[0], 2])

# Now fill `X_train_level2` with metafeatures
for cur_block_num in [27, 28, 29, 30, 31, 32, 33]:
    
    print(cur_block_num)
    
    '''
        1. Split `X_train` into parts
           Remember, that corresponding dates are stored in `dates_train` 
        2. Fit linear regression 
        3. Fit LightGBM and put predictions          
        4. Store predictions from 2. and 3. in the right place of `X_train_level2`. 
           You can use `dates_train_level2` for it
           Make sure the order of the meta-features is the same as in `X_test_level2`
    '''      
    # 根据方案 f，首先使用前 M = 15 个月的数据，因为数据从 date_block_num = 12 开始， 
    # 12 + 15 = 27, 所以循环从 27 开始，取 27 之前至 12 的数据作为测试，27 作为测试集。 
    _X_train = all_data.loc[dates <  cur_block_num].drop(to_drop_cols, axis=1) 
    _X_test  = all_data.loc[dates == cur_block_num].drop(to_drop_cols, axis=1)
    
    _y_train = all_data.loc[dates <  cur_block_num, 'target'].values
    _y_test  = all_data.loc[dates == cur_block_num, 'target'].values   
    
    
    #  linear regression
    lr.fit(_X_train.values, _y_train) # 训练
    # 预测存入 X_train_level2 即 meta-features
    # X_train_level2 中 dates_train_level2 等于当前循环月份的行的0列存入 lr 预测结果。
    
    X_train_level2[dates_train_level2 == cur_block_num, 0] = lr.predict(_X_test.values) 
    
    # LightGBM
    model = lgb.train(lgb_params, lgb.Dataset(_X_train, label=_y_train), 100)
    X_train_level2[dates_train_level2 == cur_block_num, 1] =  model.predict(_X_test)
```