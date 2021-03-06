# Implementation of Deep Speech 2 in neon

This repository contains an implementation of Baidu SVAIL's [deep speech 2] (https://arxiv.org/abs/1512.02595) model in neon. Much of the model is readily available in mainline neon, but to support the CTC cost function we have included a neon-compatible wrapper for Baidu's [Warp-CTC].
  
The deep speech 2 model is very computationally intensive. Even at near perfect GPU utilization, the model can take up to 1 week to train on large enough datasets to see respectable performance. Thus, we suggest the user keep this in mind when exploring this repo. We have used this code to train models on both the Wall Street Journal (81 hours) and Librispeech (1000 hours) datasets. The WSJ dataset is only available through the LDC, but Librispeech can be freely acquired from <http://www.openslr.org/12/>.
 
The model presented here uses a basic argmax-based decoder, simply choosing the most probable character in each frame and then collapsing the resulting output string according to CTC's rules (first remove repeat characters, then remove blank characters). With this decoding, you might expect outputs like this when trained on WSJ:

| Ground truth                    | Model output                      |
|---------------------------------|-----------------------------------|
| united presidential is a life insurance company | younited presidentiol is a lefe in surance company |
| that was certainly true last week | that was sertainly true last week |
| we're not ready to say we're in technical default a spokesman said | we're now ready to say we're intechnical default a spokesman said | 

## Getting Started
1. Make sure you have [neon] (https://github.com/NervanaSystems/neon) installed.  
    a. This repo also uses the [aeon] (https://github.com/NervanaSystems/aeon) dataloader. If you chose not to install it with neon, you must install it manually.

2. Within a neon virtualenv, run ```pip install python-levenshtein```.

3. Clone this repository: ```git clone https://github.com/NervanaSystems/deepspeech.git && cd deepspeech```

4. Run ```make``` to build warp-ctc.

## Training a model
### 1. Prepare a manifest file for your dataset.
The details on how to go about doing this are determined by the specifics of the dataset. 


#### Example: Librispeech recipe
We have provided a recipe for ingesting Librispeech data in data/ingest_librispeech.py. Note that Librispeech provides distinct datasets for training and validation, so you will need to ingest each dataset separately. Additionally, we have to get around the quirky way that the Librispeech data is distributed, so after "unpacking" the archives, we should re-pack them in a consistent manner.

To be more precise, Librispeech data is distributed in zipped tar files, e.g. `train-clean-100.tar.gz` for training and `dev-clean.tar.gz` for validation. Upon unpacking, each archive creates a directory named LibriSpeech, which means you cannot unpack both files together in the same directory. To get around this, we suggest the following. 
```
mkdir librispeech 
cd librispeech
wget http://www.openslr.org/resources/12/train-clean-100.tar.gz
wget http://www.openslr.org/resources/12/dev-clean.tar.gz
tar xvzf dev-clean.tar.gz LibriSpeech/dev-clean  --strip-components=1
tar xvzf train-clean-100.tar.gz LibriSpeech/train-clean-100  --strip-components=1
```
If you follow the above prescription, you will have the training data as a subdirectory `librispeech/train-clean-100` 
and  the validation data in a subdirectory `librispeech/dev-clean`. To ingest the data, you would then run 
```
python data/ingest_librispeech.py <absolute path to train-clean-100 directory> <absolute path to directory to write transcripts to> <absolute path to where to write training manifest to>
```

For example, if the absolute path to the train-clean-100 directory is ``/usr/local/data/librispeech/train-clean-100``, you would issue  
```
python data/ingest_librispeech.py  /usr/local/data/librispeech/train-clean-100  /usr/local/data/librispeech/train-clean-100/transcripts_dir  /usr/local/data/librispeech/train-clean-100/train-manifest.csv
```
which would create a training manifest file named train-manifest.csv. Similarly, if the absolute path to the dev-clean directory is ``/usr/local/data/librispeech/dev-clean``, you would issue  

```
python data/ingest_librispeech.py  /usr/local/data/librispeech/dev-clean  /usr/local/data/librispeech/dev-clean/transcripts_dir  /usr/local/data/librispeech/train-clean-100/val-manifest.csv
```

To train on the full 1000 hours, you should execute the same commands for the 360 hour and 540 hour training datasets as well. The manifest files can then be concatenated with a simple:
`cat /path/to/100_hour_manifest.csv /path/to/360_hour_manifest.csv /path/to/540_hour_manifest.csv > /path/to/1000_hour_manifest.csv`. 


### 2a. Train a new model

```
python train.py --manifest train:<training manifest> --manifest val:<validation manifest> -e <num_epochs> -z <batch_size> -s </path/to/model_output.pkl> [-b <backend>] 
```

where `<training manifest>` is the path to the training manifest file produced in the ingest step above (e.g. ``/usr/local/data/librispeech/train-clean-100/train-manifest.csv`` in the example above) and `<validation manifest>` is the path to the validation manifest file.
 
### 2b. Continue training a previous model
If you have a previously trained model, you can resume training by passing the `--model_file </path/to/stored_model.pkl>` argument to `train.py`. Soon we will provide a model trained on the 1000 hour Librispeech dataset and will provide details on how to download and use the model at that time. 

## Decoding and evaluating a trained model
Once you have a trained model, you can easily evaluate its performance on any given dataset. Simply create a manifest file and then call:
 ```
 python evaluate.py --manifest val:/path/to/manifest.csv --model_file /path/to/saved_model.pkl
 ```
Replacing the file paths as needed. This will print out character error rates by default. To print word error rates, include the argument `--use_wer`.

[Warp-CTC]: https://github.com/baidu-research/warp-ctc
