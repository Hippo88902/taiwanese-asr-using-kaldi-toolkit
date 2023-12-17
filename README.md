# 學號: 311511057 姓名: 張詔揚

# 使用Kaldi & ESPnet來做台語語音辨認

資料規格:

1. 單人女聲聲音（高雄腔）
2. 輸入：台語語音音檔（格式：wav檔, 22 kHz, mono, 32 bits） 
3. 輸出：台羅拼音（依教育部標準）

- **Kaggle的材料**
    1. **`train`**
        - 4 hour 15 min
        - 可以從 train 切一些資料來 eval 用
    2. **`test`**
        - 28 min
    3. **`train-toneless.csv`**
    4. **`lexicon.txt`**
    5. **~~sample.csv　（參考用）~~**
    6. **~~train.csv　（用不到）~~**
  
事前準備:

1. 在server上安裝kaldi
2. 從kaggle上下載data的壓縮檔
3. 將data放入kaldi/egs目錄下解壓縮

## Table of Contents
- [Kaldi-enviroment-setting](#Kaldi-enviroment-setting)
- [Data-Preprocessing-for-Kaldi](#Data-Preprocessing-for-kaldi)
- [Shell-script調整-Kaldi](#shell-script調整-kaldi)
- [Training-Kaldi](#training-kaldi)
- [使用ESPnet做台語語音辨認](#使用espnet做台語語音辨認)
- [Data-Preprocessing-for-ESPnet](#data-preprocessing-for-espnet)
- [Training-ESPnet](#Training-ESPnet)
- [Conclusion](#conclusion)

## Kaldi-enviroment-setting
Kaldi環境安裝

1. 下載kaldi
```sh
$ git clone https://github.com/kaldi-asr/kaldi.git
```
2. 在/kaldi/tools中安裝第三方工具、語言模型
```sh
$ make 
$ extras/install_irstlm.sh
$ extras/install_mkl.sh # 這個一定要裝起來，不然跑不了，有bug要Debug
```
3. 在tools安裝 srilm 語言模型
   a. 下載srilm的檔案
   b. 改名成srilm.tgz，放到tools裡面
   c. 安裝
 ```sh
 $ ./install_srilm.sh
 $ ./extras/install_kaldi_lm.sh
 $ source env.sh
 ```
4. 到tools編譯
```sh
$ make -j 32
```
- 32為電腦的**CPU核心數**，可用`cat /proc/cpuinfo | grep "processor" | wc -l`來查看
- `-j`是一個並行處理機制，可以加快編譯的速度
- 如果正確執行會顯示 All done OK.
5. 到src裡建置source code
```sh
$ ./configure --shared 
$ make depend -j 32 
$ make -j 32
```
安裝好kaldi後，可以到egs/yesno目錄測試kaldi是否正確安裝。
```sh
# cd到 egs/yesno/s5 輸入
$ bash run.sh 
# 或是
$ ./run.sh
```
若有score的結果則代表有成功執行。

Important directories for Kaldi：

1. tools　(存放安裝工具、語言模型)
2. src　（儲存source code）
3. egs　（儲存開源的不同的資料集的shell scripts）

按照上述步驟可以建立kaldi的環境，但環境設置會因為每一個人電腦、OS的不同而出現不同的問題，請依照terminal顯示的內容安裝對應套件。

**參考資料：**

https://kaldi-asr.org/doc/index.html

https://blog.csdn.net/DL_62532/article/details/128156452

## Data-Preprocessing-for-Kaldi

下載原始資料後，需要先對資料進行前處理，否則無法使用kaldi進行訓練。

The steps for data preprocessing:

1. 將trian.csv以及test.csv檔處理成kaldi可接受的格式
2. 由於音檔格式為（wav檔, 22 kHz, mono, 32 bits），因此使用 sox 將音檔轉成（wav檔, 16 kHz, mono, 16 bits）
，train、test的音檔請先用sox來轉成 16 kHz sampling, signed-integer, 16 bits，可以直接cd到音檔資料夾輸入下面這個指令，或是建立一個.sh檔案執行
   ```
   #!/bin/bash
   for x in ./*.wav;do
   b=${x##*/}
   sox $b -r 16000 -e signed-integer -b 16 tmp-$b
   rm -rf $b
   mv tmp-$b $b
   done
   ​
可以用sox來看音檔的資訊
`soxi <audiofile>`
3. 在s5目錄下，建立lexicon資料夾，並將kaggle上的lexicon檔案放進去

![image](https://github.com/MachineLearningNTUT/taiwanese-asr-using-kaldi-toolkit-Hippo88902/blob/main/s5.jpg)

****Kaldi recipes的架構****

直接從`formosa`複製整個資料夾，取一個新的名字，看你想要取什麼，我取`taiwanese`(可以自己換)

- egs每一個 **recipe** 的`steps`, `utils`都共用，都會link到`wsj`下面的，所以可以直接複製整個formosa的資料夾即可

現在`taiwanese/s5`下面有的檔案(只列重要的)：

1. **conf　（不用動）**
2. **local　（很重要）**
    1. **prepare_data.sh**：把資料換成kaldi可以吃的格式
    2. **prepare_dict.sh**：給kaldi我們的字典
    3. **nnet3**：kaldi的**DNN框架**
    4. **chain**：結合nnet3的一種新的方法，目前**效果**最好的
3. **steps　（不用動）**
4. **utils　（不用動）**
5. **cmd.sh　（把queue.pl改成run.pl）**
6. **path.sh　（有問題再動）**
7. **run.sh　（主要執行的腳本）**

## Shell-script調整-Kaldi

在進行訓練前，還必須對shell script (cmd.sh、 path.sh、 run.sh ...) 進行一些調整，才能夠順利進行training。

Tips:

1. 執行程式後發生錯誤時，多查看.log檔，可節省大量偵錯時間
2. 找不出錯時，可對照原始檔的設定，比較一下與我們的data有什麼不同，有時便能豁然開朗
3. 確認shell-script裡所有的目錄都有放對地方，很常發生這種錯誤

p.s.: 改shell script的步驟很單調，也很無聊，不過成功之後就能夠順利進行training了

## Training-Kaldi

1. 第一階段，data preparation並設定number of jobs，以及train language model。
2. 第二階段，求MFCC features，以及tri1~tri5(對齊並解碼)。
3. 第三階段，選擇我們要的model開始training，有nnet3 tdnn models 以及 chain model兩種可選，通常chain model所花時間較短，且結果較好。

```sh
$ sudo nvidia-smi -c 3
# 讓gpu進入獨佔模式，可加快訓練的速度(不過要先跟其他人協調好再下這行指令)
$ nohup ./run.sh >& run.sh.log &
# 保證登出不會中斷執行程式，因為training時間較久，下這個指令能確保訓練過程不會因為突發情況中斷。
```
p.s.: 如果過程順利，就只要等training結束，若訓練中途出錯，則可根據.log檔去debug。

## 使用ESPnet做台語語音辨認

也可以使用ESPnet來做ASR，與kaldi不同，ESPnet是使用類神經網路模型，因此在訓練前，不需要跟kaldi一樣，
使用MFCC去求切割位置，而是利用深度學習的方式去訓練。

事前準備:

1. 在server上安裝ESPnet (建議新建一個虛擬環境)
2. 確認各種驅動程式，以及套件的版本與ESPnet是相容的 (ex: pytorch、cuda...)
3. 由於有些shell script裡會使用到kaldi，需事先將soft link建立好，否則會出錯

## Data-Preprocessing-for-ESPnet


1. 與Kaldi不同，ESPnet除train/test外，另外還需一組dev資料來幫助訓練，使ESPnet在training時，能及時協助評估訓練效能。此外我們通常會將dataset分割為train:test:validation三個部分，三者比例分別為8:1:1，並將所有資料放在downloads目錄裡。


2. 在downloads/resource_aishell裡放入speaker.info和lexicon，兩者分別為語者性別及與之對應的辭典。

3. 因為資料集為單語者資料，所以可以將所有訓練資料都放在downloads/data_aishell/wav/train/global裡面，若資料為多語者資料，可在download/data_aishell/wav/train裡面依每個語者編號順序放置訓練資料的.wav檔。

## Training-ESPnet

1. 第一階段，特徵抽取(Feature extraction)，首先求fbank,並讓80%的fbank在每一frame都有pitch(訓練用)，接著做speed-perturbed 為了 data augmentation，以及求global CMVN(在某個範圍内統計若干聲學特徵的mean和variance)，如果提供了語句和語者的對應關係，即uut2spk，則進行speaker CMVN，否則做global CMVN。由於我們的資料是單語者資料，所以做global CMVN。

2. 第二階段，準備字典(Dictionary Preparation)，對訓練資料所有出現的word進行處理，為了使lexicon與wav音檔的答案對應。

3. 第三階段，準備語言模型(LM preparation) ，訓練我們的語言模型，並產生n-gram model(這裡我們用4-gram)。

4. 第四階段，Network training，最花時間也最耗費GPU計算資源的階段，訓練的過程會藉由dev的辨識結果進行修正，讓model能夠越訓練效果越好。

5. 最終階段，Decode完後，即可得到我們要的ASR model，並辨識test裡面的.wav檔案，並得到訓練的結果。

![image](https://github.com/Hippo88902/taiwanese-asr-using-kaldi-toolkit/blob/main/%E8%A8%93%E7%B7%B4%E7%B5%90%E6%9E%9C.png)

p.s.1: 前三個階段可看作是training的事前準備，都交由CPU去處理，因此所花費的時間並不多。

p.s.2: 除此之外，也能調整batch size大小，讓GPU的memory能夠盡量地吃滿(提高利用度)，此舉能夠加快training的速度。

## Conclusion

![image](https://github.com/MachineLearningNTUT/taiwanese-asr-using-kaldi-toolkit-Hippo88902/blob/main/Kaldi%E7%B5%90%E6%9E%9C.jpg)

![image](https://github.com/MachineLearningNTUT/taiwanese-asr-using-kaldi-toolkit-Hippo88902/blob/main/ESPnet%E7%B5%90%E6%9E%9C.jpg)

總體來說，ESPnet的訓練效果會比Kaldi的訓練效果來的好，不過ESPnet所需要的訓練時間也比較長，也較耗費計算資源。然而在資料量較少的情況下，Kaldi基於傳統機率統計模型的訓練方式，或許會表現的比ESPnet還好，但是當data的量足夠多時，神經網路的訓練方式通常會outperform傳統的機率統計模型。因此在資料量充足時，我們會採用Neural network的方式進行訓練，在資料量不足以train起一個model時，則可以使用機率統計模型來得到一個還不差的結果，這或許成為了一種data數量上的trade-off。
