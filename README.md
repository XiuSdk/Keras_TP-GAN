# Keras_TP-GAN

## Unofficial Keras (with Tensorflow) re-implementation of "Beyond Face Rotation: Global and Local Perception GAN for Photorealistic and Identity Preserving Frontal View Synthesis"
Main part of this code is implemented referring to the author's official pure Tensorflow implementation.

https://github.com/HRLTY/TP-GAN

Original paper is

*Huang, R., Zhang, S., Li, T., & He, R. (2017). Beyond face rotation: Global and local perception gan for photorealistic and identity preserving frontal view synthesis. arXiv preprint arXiv:1704.04086.*

https://arxiv.org/abs/1704.04086

## Current results
Currently, generalization and resolution are not good as the author's results. If you have some idea, I'm very happy if I have a discussion with you. Any comments are welcomed.

Input | Synthesized frontal | True frontal
----|----|----
TD1 | TD3 |
TD2 | TD4 |
TD2 | TD4 |
TD2 | TD4 |


## Library version
- Python: 3.6.3
- Tensorflow: 1.5.0
- Keras: 2.1.3
- GPU: GeForce GTX 1080 Ti (single)
- TP-GAN is Developed on Anaconda 5.0.1, Windows10 64bit
- Face landmarks extraction is executed on CentOS 7

## Building dataset
- Purchase MULTI-PIE dataset from
http://www.cs.cmu.edu/afs/cs/project/PIE/MultiPie/Multi-Pie/Home.html.
- Convert MULTI-PIE target images into jpeg with same directory structures. misc/jpeg_converter.py can help this.
- Extract landmarks as np.array from target images and store them into pickled dictionary file (landmarks.pkl). misc/extract_from_multipie.py and misc/landmark_convert.py can help this.
In this experiments, I used following python module on CentOS 7 for landmark extraction, though all other process are done in Windows 10.
https://github.com/1adrianb/face-alignment

## Fine-tuning LightCNN model
First, train Light CNN with MS-Celeb-1M 'ALIGNED' dataset with cleaned list. In this experiments I used Light CNN code at https://github.com/yh-iro/Keras_LightCNN. Please see this instructions.

Second, with trained extractor fixed, train new classifier with MULTI-PIE.

Third, fine-tune the extractor with MULTI-PIE.

Followings are code examples of second and third steps.

```python
from keras_tpgan.tpgan import LightCNN, multipie_gen
from keras.optimizers import SGD

datagen = multipie_gen.Datagen(valid_count=300, min_angle=-15, max_angle=15,
                               include_frontal=True, face_trembling_range=8, mirror_to_one_side=False)
lcnn = LightCNN(classes=datagen.get_subjects(),
                extractor_weights='TRAINED/Keras_LightCNN/EXTRACTOR/MODEL/WITH/MS-CELEB-1M.hdf5',
                classifier_weights=None)

# second step
train_gen = datagen.get_class_generator(setting='train', batch_size=64)

lcnn.train(train_gen=train_gen, valid_gen=train_gen, optimizer=SGD(lr=0.001, momentum=0.9, decay=0.00004, nesterov=True),
                     classifier_dropout=0.5, steps_per_epoch=1000, validation_steps=50,
                     epochs=25, out_dir='OUT_DIR/', out_period=5, fix_extractor=True)

# third step
lcnn.train(train_gen=train_gen, valid_gen=train_gen, optimizer=SGD(lr=0.0001, momentum=0.9, decay=0.00004, nesterov=True),
                     classifier_dropout=0.5, steps_per_epoch=1000, validation_steps=50,
                     epochs=25, out_dir='OUT_DIR/', out_period=5, fix_extractor=False)
```

I trained 1060 epochs for the first step, 50 epochs for the second step, and 50 epochs for the third step.
After fine-tuning, the trained model scores 0.997 as loss value, 1.000 as valacc.

## Training
My training scripts history resulting to the sample pictures on top of this document. Note that it includes try and errors and still updating thus not the best. In this example, I called my python module through Google ml-engine from Windows command prompt.

```
gcloud ml-engine local train --module-name keras_tpgan.tasks.train_gan --package-path keras_tpgan/ ^
-- ^
--out_dir "OUT_DIR/" ^
--gpus 1 ^
--lcnn-weights "FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--datalist_dir "DATALIST_DIR/" ^
--gen_batch_size 8 ^
--gen_steps_per_epoch 1000 ^
--disc_batch_size 16 ^
--disc_steps_per_epoch 100 ^
--epochs 30 ^
--out_period 3 ^
--optimizer adam ^
--lr 0.0001 ^
--decay 0 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: 5e-3" ^
--lambda_tv "lambda x: 1e-5" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3" >> OUT_DIR/train.log 2>&1
```

```
gcloud ml-engine local train --module-name keras_tpgan.tasks.train_gan --package-path keras_tpgan/ ^
-- ^
--out_dir "OUT_DIR/" ^
--gpus 1 ^
--lcnn-weights "FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--generator-weights "OUT_DIR/weights/generator/epoch0030_loss1.604.hdf5" ^
--classifier_weights "OUT_DIR/weights/classifier/epoch0030_loss1.604.hdf5" ^
--discriminator_weights "OUT_DIR/weights/discriminator/epoch0030_loss0.163.hdf5" ^
--datalist_dir "datalist/" ^
--gen_batch_size 8 ^
--gen_steps_per_epoch 1000 ^
--disc_batch_size 16 ^
--disc_steps_per_epoch 100 ^
--epochs 90 ^
--out_period 3 ^
--optimizer adam ^
--lr 0.0001 ^
--decay 0 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: (5e-3*(1+2*(x/5))) if x<100 else 5e-3*3" ^
--lambda_tv "lambda x: 1e-5" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3" >> OUT_DIR/train.log 2>&1
```

```
gcloud ml-engine local train --module-name keras_tpgan.tasks.train_gan --package-path keras_tpgan/ ^
-- ^
--out_dir "OUT_DIR/" ^
--gpus 1 ^
--lcnn-weights "FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--generator-weights "OUT_DIR/weights/generator/epoch0120_loss1.062.hdf5" ^
--classifier_weights "OUT_DIR/weights/classifier/epoch0120_loss1.062.hdf5" ^
--discriminator_weights "OUT_DIR/weights/discriminator/epoch0120_loss0.177.hdf5" ^
--datalist_dir "datalist/" ^
--gen_batch_size 8 ^
--gen_steps_per_epoch 1000 ^
--disc_batch_size 16 ^
--disc_steps_per_epoch 100 ^
--epochs 12 ^
--out_period 3 ^
--optimizer adam ^
--lr 0.0001 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: (1.5e-2*(1+2*(x/50))) if x^<50 else 1.5e-2*3" ^
--lambda_tv "lambda x: 1e-5" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3" >> OUT_DIR/train180514.log 2>&1
```

```
gcloud ml-engine local train --module-name keras_tpgan.tasks.train_gan --package-path keras_tpgan/ ^
-- ^
--out_dir "OUT_DIR/" ^
--gpus 1 ^
--lcnn-weights "FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--generator-weights "OUT_DIR/weights/generator/epoch0132_loss1.006.hdf5" ^
--classifier_weights "OUT_DIR/weights/classifier/epoch0132_loss1.006.hdf5" ^
--discriminator_weights "OUT_DIR/weights/discriminator/epoch0132_loss0.288.hdf5" ^
--datalist_dir "datalist/" ^
--gen_batch_size 8 ^
--gen_steps_per_epoch 300 ^
--disc_batch_size 16 ^
--disc_steps_per_epoch 100 ^
--epochs 60 ^
--out_period 3 ^
--optimizer adam ^
--lr 0.0001 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: (2.156e-2*(1+2*(x/50))) if x^<50 else 2.156e-2*3" ^
--lambda_tv "lambda x: 1e-5" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3" >> OUT_DIR/train180514.log 2>&1
```

```
gcloud ml-engine local train --module-name keras_tpgan.tasks.train_gan --package-path keras_tpgan/ ^
-- ^
--out_dir "OUT_DIR/" ^
--gpus 1 ^
--lcnn-weights "FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--generator-weights "OUT_DIR/weights/generator/epoch0192_loss0.979.hdf5" ^
--classifier_weights "OUT_DIR/weights/classifier/epoch0192_loss0.979.hdf5" ^
--discriminator_weights "OUT_DIR/weights/discriminator/epoch0192_loss0.234.hdf5" ^
--datalist_dir "datalist/" ^
--gen_batch_size 8 ^
--gen_steps_per_epoch 500 ^
--disc_batch_size 16 ^
--disc_steps_per_epoch 100 ^
--epochs 1000 ^
--out_period 3 ^
--optimizer adam ^
--lr 0.0001 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: 6.5e-2" ^
--lambda_tv "lambda x: (1e-5*(1+9*(x/100))) if x^<100 else 1e-5*10" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3" >> OUT_DIR/train180514.log 2>&1
```


## Using Google ML Engine and Storage
Training environment can easily be switched to [Google Cloud Machine Learning Engine](https://cloud.google.com/ml-engine/).
- Before use it, you need to upload MULTI-PIE to Google cloud storage. There are several way to upload images, but using python module is the fastest for my environment. misc/uploader.py can help this. In my case, it requires about 3 days to upload images used for training (not whole MULTI-PIE images).
- Though Google ML Engine has multiple high-spec gpus, in my case, file IO was critical bottle neck even though I implemented easy multi-threading.
- Training scripts example on Windows command prompt (but not maintained well)

```
cd PATH/TO/Keras_TP-GAN

gcloud ml-engine jobs submit training JOB_ID ^
--module-name keras_tpgan.tasks.train_gan ^
--job-dir "gs://BUCKET_NAME/JOB-DIR/" ^
--package-path "keras_tpgan/" ^
--region asia-east1 ^
--config "config.yaml" ^
-- ^
--multipie_dir "gs://BUCKET_NAME/images/" ^
--landmarks_dict_file "gs://BUCKET_NAME/landmarks.pkl" ^
--datalist_dir "gs://BUCKET_NAME/apps/TPGAN/datalist/" ^
--out_dir "OUT_DIR/" ^
--gpus 4 ^
--lcnn-weights "gs://BUCKET_NAME/FINE-TUNED/Keras_LightCNN/EXTRACTOR/MODEL.hdf5" ^
--generator-weights "gs://BUCKET_NAME/BUCKET_NAME/JOB-DIR/OUT_DIR/weights/generator/WEIGHTS.hdf5" ^
--classifier_weights "gs://BUCKET_NAME/BUCKET_NAME/JOB-DIR/OUT_DIR/weights/classifier/WEIGHTS.hdf5" ^
--discriminator_weights "gs://BUCKET_NAME/BUCKET_NAME/JOB-DIR/OUT_DIR/weights/discriminator/WEIGHTS.hdf5" ^
--gen_batch_size 32 ^
--gen_steps_per_epoch 1000 ^
--disc_batch_size 64 ^
--disc_steps_per_epoch 100 ^
--epochs 100 ^
--out_period 1 ^
--optimizer adam ^
--lr 0.0001 ^
--decay 0 ^
--beta1 0.9 ^
--lambda_128 "lambda x: 1" ^
--lambda_64 "lambda x: 1" ^
--lambda_32 "lambda x: 1.5" ^
--lambda_sym "lambda x: 1e-1" ^
--lambda_ip "lambda x: 1e-3" ^
--lambda_adv "lambda x: 6.5e-2" ^
--lambda_tv "lambda x: (1e-5*(1+9*(x/100))) if x^<100 else 1e-5*10" ^
--lambda_class "lambda x: 1" ^
--lambda_parts "lambda x: 3"
```