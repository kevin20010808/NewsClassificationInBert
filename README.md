
:::info
:bulb: 使用agnews的作業範本，使用環境為GPU 3060ti。
:::

## 🎉 基礎實現
:::success
依照樣本的內容實現使用bert來fine tune，結果可以取得準確率0.91-0.94的好成績。
:::
### :small_orange_diamond:遇到的問題：
- [ ] 訓練時間過長：14分34秒。
- [ ] 與大多實際輸入的cases不準確，可能是過擬和導致。

## :book: LSTM復現
:::success
嘗試實現LSTM，作為NLP的基礎練習。
:::
### :small_orange_diamond:結果:
- [ ] 訓練時間大幅降低，使用epoch=5：共約1分45秒。
- [ ] 準確率下降：0.90。
- [ ] 自己輸入的cases也可能錯誤，但機率不高。應該是準確率不夠導致。

## :white_check_mark: BiLSTM+attention復現

:::success
嘗試用BiLSTM+attention改進，也作為NLP的基礎練習。
:::
### :small_orange_diamond:結果:
- [ ] 訓練時間沒有變長太多，使用epoch=5：共3分13秒。
- [ ] 準確率提高：0.91。
- [ ] 自己輸入的cases幾乎沒有出錯，比較沒有過擬和的傾向。


## :fire: Further Attempt
### 1. Lora
我們不需要對整個權重矩陣進行完整的更新，而只需要更新一個低秩的近似矩陣

[參考論文 — LORA: LOW-RANK ADAPTATION OF LARGE LANGUAGE MODELS](https://arxiv.org/pdf/2106.09685)

### :small_blue_diamond: 問題與方法：
當一個很大的pretrain model要重新training中fine tune所有的parameters，是不太可能的。因此Low-Rank可以凍結一些weight，把trainable的decomposited matrices放到transformer的每一層，減少parameters。
![image](https://hackmd.io/_uploads/SJaknqTqJl.png)
從圖中可以看到比起訓練藍色，只訓練橘色部分會減少很多參數量。這是從另一篇「過度參數化」的論文得到的猜想。
``
大概是指龐大的參數中，模型在新任務中需要的權重變化可能只會在低維度的子空間中。即根本不用考慮所有參數。
``

### :small_blue_diamond: 優勢：
* 因為不會替換掉原始預訓練模型的parameters，所以可以在不同任務上替換。有效率的切換任務。
* 不用計算梯度或維護optimizer，可以降低硬體門檻。(只需要維持橘色部分)
* 基於線性變換，在推理時不會出現延遲。


### :small_blue_diamond: 方法：

* 初始化: A 初始化為隨機高斯分佈。B 初始化為零矩陣，這樣一開始不會影響模型的原始行為。訓練開始時，模型的權重變化為零，然後隨著訓練進行，A 和 B 開始學習適應特定任務的更新。
* R可以決定參數量，如上圖，也可以放寬成全參數。alpha/r來縮放BAx，當使用Adam進行最佳化時，如果適當縮放初始化，則調整α與調整學習率大致相同。因此，我們只需將α設定為嘗試的第一個r，而不進行調整。當改變r時，可以減少重新調整超參數的需要。
* 為了簡化並提高參數效率，LoRA 只作用於**自注意力機制的權重矩陣**，而MLP層則不參與訓練。

### :small_orange_diamond:結果
:::success
有出現顯著的顯存下降，參數減少。甚至有增加效能。
:::
- [ ] 只關注在減少要記憶的參數，但仍需要計算整個model的梯度。時間稍微縮短至11分鐘。
- [ ] 主要節省了顯示卡的儲存空間，降低了硬體需求。
- [ ] Bert: Allocated: 1.66 GB, Reserved: 4.07 GB
- [ ] Lora: Allocated: 0.81 GB, Reserved: 2.78 GB。使用率減少一半。
- [ ] LoRA 可以看出參數下降很多 Total Params: 109,632,772, Trainable Params: 147,456。約剩0.1%
- [ ] 準確率:0.91，幾乎沒有下降。但學習率部分有調高。而且random cases似乎更準確，沒有過擬和發生。

### 2. Weighted voting
:::success
使用表現較好的BiLSTM+attention。以及，輕便後的Lora+bert。
:::
將兩個模型都跑一次，訓練時間就是兩個相加。然後按照權重分配去投票。
### :small_orange_diamond:結果:
- [ ] 訓練時間為兩個模型的訓練時間和。
- [ ] 準確率微幅提升。兩個模型的準確率都在0.916-0.90之間，穩定提升至0.925左右。但過於微幅。
- [ ] random cases都準確。可能是任務較為簡易的分類任務，原本目標預期可以提升到0.94以上，但似乎遇到瓶頸。

## :clock1: 下個進度(maybe)
### :small_orange_diamond:研究別人的Kaggle方法(如何用LSTM強化BERT、Pearson Loss、Linear Attention Pooling)
https://www.kaggle.com/competitions/titanic
### :small_orange_diamond:Adversarial Weight Perturbation 
https://www.kaggle.com/code/itsuki9180/introducing-adversarial-weight-perturbation-awp/notebook


## :bulb: Meeting Feedback
"錯誤分析"
運用confusion matrix下去思考
或是查看cls token的效果
可以先用input PCA做降維 分析為甚麼兩篇文章丟進去出來的會不一樣
用bert通常不會接lora，bert下面可以接更多東西來做變化。(線性層、BiLSTM、CRF、)

## AgNews作業重作

1. AutoModelForSequenceClassification(baseline)
Classification Report:
               
               precision    recall  f1-score   support

       World        0.93      0.95      0.94      5956
       Sports       0.99      0.97      0.98      6058
       Business     0.92      0.89      0.91      5911
       Tech         0.91      0.94      0.92      6075


Macro F1 Score: 0.936 

![image](https://hackmd.io/_uploads/BkvB_2m0yl.png)



2. bert+fc

Classification Report:

                    precision    recall  f1-score   support

       World          0.93      0.92      0.93      5956
       Sports         0.97      0.97      0.97      6058
       Business       0.87      0.92      0.90      5911
       Sci/Tech       0.92      0.88      0.90      6075
       
Macro F1 Score: 0.924

![image](https://hackmd.io/_uploads/SJwc23GA1l.png)

3. bert+lstm

Classification Report:

                 precision    recall  f1-score   support
       World        0.96      0.93      0.95      5956
       Sports       0.98      0.99      0.99      6058
       Business     0.93      0.85      0.89      5911
       Tech         0.86      0.96      0.91      6075

Macro F1 Score: 0.933


![image](https://hackmd.io/_uploads/H1Iqq9mAye.png)



4. bert+TextCNN

Classification Report:
             
                precision    recall  f1-score   support

       World       0.95      0.93      0.94      5956
      Sports       0.98      0.98      0.98      6058
    Business       0.92      0.89      0.91      5911
        Tech       0.90      0.94      0.92      6075

Macro F1 Score: 0.937

![image](https://hackmd.io/_uploads/SJxUMcQ0yx.png)



5. bert+MLP+Residual

Classification Report:
              
              precision    recall  f1-score   support

       World       0.95      0.94      0.95      5956
      Sports       0.99      0.98      0.98      6058
    Business       0.92      0.90      0.91      5911
        Tech       0.90      0.94      0.92      6075

Macro F1 Score: 0.9389

![image](https://hackmd.io/_uploads/B1Sf8j7Rkl.png)

### 其他嘗試
1. embedding分析:
從CLS使用PCA和T-SNE降維和可視化，目的是想找出為什麼Business和Tech這麼難分類。
![image](https://hackmd.io/_uploads/Bkes527A1g.png)

![image](https://hackmd.io/_uploads/Skcj92XR1l.png)
從PCA的分析上看，除了Sports以外三類都很多穿插在不同類的點，對於world和business可能還好，尤其Tech就很容易被分類到旁邊兩類，尤其Tech和Business還有一堆點幾乎融在一起。因此Tech類也是錯誤最多的。

2. 錯詞觀察:
我嘗試從原文使用TF-IDF列出最容易被搞混的詞。
Business容易被誤分類為Tech的詞:
['words', 'important', 'headline', 'internet', 'al', 'com', 'search', 'online', 'service', 'technology', 'microsoft', 'lt', 'web', '39', 'ap']
 
可以看出形容詞的占比很大，因為對類別沒有直接語意上的關聯。還有科技公司，本來就存在商業和科技的性質，不能作為分類依據。還有數字，也是這兩個板塊容易出現的。
3. Attention調整:
因為這些字根本不能作為分類依據，我想將這些字從模型中降低一些關注以免誤導。因此我嘗試:
* 能不能把這些詞弄成一個清單，懲罰這些詞的注意力分數。所以先把這些詞轉成tokenize，如果input_ids一樣就扣分。
* 那我bert後面可以接一個attention pooling層，然後懲罰機制，再去softmax非線性處理梯度太低問題，這樣扣分就可以扣重一點。


實驗結果: 失敗


我用表現最好的bert+MLP+Residual 中間加上設計好的attention pooling
變得很傾向於猜business，Business → Tech 錯誤大減，但更糟的是，Tech → Business 錯誤暴增。

Macro F1 Score: 0.9207

![image](https://hackmd.io/_uploads/H1_nPpQCJe.png)

為何沒用?
有可能是，我只懲罰了猜錯tech但沒有懲罰猜錯business，所以產生偏移。
那兩邊都懲罰呢? 我將同時懲罰兩邊各前15個易誤判的詞。
Classification Report:
              
              precision    recall  f1-score   support

       World       0.95      0.94      0.95      5956
      Sports       0.98      0.98      0.98      6058
    Business       0.92      0.89      0.91      5911
        Tech       0.90      0.94      0.92      6075

Macro F1 Score: 0.9394
依然沒用:我嘗試增加懲罰的字。

![image](https://hackmd.io/_uploads/r1X2y0QAJg.png)


![image](https://hackmd.io/_uploads/rJE0lR7Ckl.png)

增加後結果反而下降了，也就是說可能刪掉了具有語意的字，造成學習不夠，模型猜對的機率變低。

因此考慮其他方法可能才是更好的：
未來加入EDA或增加SHAP分析。
