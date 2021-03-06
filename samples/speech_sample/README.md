# Automatic Speech Recognition Sample {#InferenceEngineASRApplication}

This topic shows how to run the speech sample application, which
demonstrates acoustic model inference based on Kaldi neural networks
and speech feature vectors.

## Running

### Usage

Running the application with the `-h` option yields the following
usage message:

```sh
$ ./speech_sample -h
InferenceEngine: 
    API version ............ <version>
    Build .................. <number>

speech_sample [OPTION]
Options:

    -h                      Print a usage message.
    -i "<path>"             Required. Path to an .ark file.
    -m "<path>"             Required. Path to an .xml file with a trained model (required if -rg is missing).
    -o "<path>"             Output file name (default name is scores.ark).
    -l "<absolute_path>"    Required for MKLDNN (CPU)-targeted custom layers.Absolute path to a shared library with the kernels impl.
    -d "<device>"           Specify the target device to infer on; CPU, GPU, GNA_AUTO, GNA_HW, GNA_SW, GNA_SW_EXACT is acceptable. Sample will look for a suitable plugin for device specified
    -p                      Plugin name. For example MKLDNNPlugin. If this parameter is pointed, the sample will look for this plugin only
    -pp                     Path to a plugin folder.
    -pc                     Enables performance report
    -q "<mode>"             Input quantization mode:  static (default), dynamic, or user (use with -sf).
    -qb "<integer>"         Weight bits for quantization:  8 or 16 (default)
    -sf "<double>"          Optional user-specified input scale factor for quantization (use with -q user).
    -bs "<integer>"         Batch size 1-8 (default 1)
    -r "<path>"             Read reference score .ark file and compare scores.
    -rg "<path>"            Read GNA model from file using path/filename provided (required if -m is missing).
    -wg "<path>"            Write GNA model to file using path/filename provided.
    -we "<path>"            Write GNA embedded model to file using path/filename provided.

```

Running the application with the empty list of options yields the
usage message given above and an error message.

### Model Preparation

You can use the following model optimizer command to convert a Kaldi
nnet1 or nnet2 neural network to Intel IR format:

```sh
$ python3 mo.py --framework kaldi --input_model wsj_dnn5b_smbr.nnet --counts wsj_dnn5b_smbr.counts --remove_output_softmax
```

Assuming that the model optimizer (`mo.py`), Kaldi-trained neural
network, `wsj_dnn5b_smbr.nnet`, and Kaldi class counts file,
`wsj_dnn5b_smbr.counts`, are in the working directory this produces
the Intel IR network consisting of `wsj_dnn5b_smbr.xml` and
`wsj_dnn5b_smbr.bin`.

Note that `wsj_dnn5b_smbr.nnet` and other sample Kaldi models and
data will be available in July 2018 in the OpenVINO Open Model Zoo.

### Speech Inference

Once the IR is created, you can use the following command to do
inference on Intel^&reg; Processors with the GNA co-processor (or
emulation library):

```sh
$ ./speech_sample -d GNA_AUTO -bs 2 -i wsj_dnn5b_smbr_dev93_10.ark -m wsj_dnn5b_smbr_fp32.xml -o scores.ark -r wsj_dnn5b_smbr_dev93_scores_10.ark
```

Here, the floating point Kaldi-generated reference neural network
scores (wsj_dnn5b_smbr_dev93_scores_10.ark) corresponding to the input
feature file (wsj_dnn5b_smbr_dev93_10.ark) are assumed to be available
for comparison.

### Sample Output

The acoustic log likelihood sequences for all utterances are stored in
the Kaldi ARK file, scores.ark.  If the `-r` option is used, a report on
the statistical score error is generated for each utterance such as
the following:

``` sh
Utterance 0: 4k0c0301
   Average inference time per frame: 6.26867 ms
         max error: 0.0667191
         avg error: 0.00473641
     avg rms error: 0.00602212
       stdev error: 0.00393488
```

## How it works

Upon the start-up the speech_sample application reads command line parameters
and loads a Kaldi-trained neural network along with Kaldi ARK speech
feature vector file to the Inference Engine plugin. It then performs
inference on all speech utterances stored in the input ARK
file. Context-windowed speech frames are processed in batches of 1-8
frames according to the -bs parameter.  Batching across utterances is
not supported by this sample.  When inference is done, the application
creates an output ARK file.  If the `-r` option is given, error
statistics are provided for each speech utterance as shown above.

### GNA-specific details

#### Quantization

If the GNA device is selected (for example, using the `-d` GNA_AUTO flag),
the GNA Inference Engine plugin quantizes the model and input feature
vector sequence to integer representation before performing inference.
Several parameters control neural network quantization.  The `-q` flag
determines the quantization mode.  Three modes are supported: static,
dynamic, and user-defined.  In static quantization mode, the first
utterance in the input ARK file is scanned for dynamic range.  The
scale factor (floating point scalar multiplier) required to scale the
maximum input value of the first utterance to 16384 (15 bits) is used
for all subsequent inputs.  The neural network is quantized to
accommodate the scaled input dynamic range.  In user-defined
quantization mode, the user may specify a scale factor via the `-sf`
flag that will be used for static quantization.  In dynamic
quantization mode, the scale factor for each input batch is computed
just before inference on that batch.  The input and network are
(re)quantized on-the-fly using an efficient procedure.

The `-qb` flag provides a hint to the GNA plugin regarding the preferred
target weight resolution for all layers.  For example, when `-qb 8` is
specified, the plugin will use 8-bit weights wherever possible in the
network.  Note that it is not always possible to use 8-bit weights due
to GNA hardware limitations.  For example, convolutional layers always
use 16-bit weights (GNA harware verison 1 and 2).  This limitation
will be removed in GNA hardware version 3 and higher.

#### Execution Modes

Several execution modes are supported via the `-d` flag.  If the device
is set to CPU and the GNA plugin is selected, the GNA device is
emulated in fast-but-not-bit-exact mode.  If the device is set to
GNA_AUTO, then the GNA hardware is used if available and the driver is
installed.  Otherwise, the GNA device is emulated in
fast-but-not-bit-exact mode.  If the device is set to GNA_HW, then the
GNA hardware is used if available and the driver is installed.
Otherwise, an error will occur.  If the device is set to GNA_SW, the
GNA device is emulated in fast-but-not-bit-exact mode.  Finally, if
the device is set to GNA_SW_EXACT, the GNA device is emulated in
bit-exact mode.

#### Loading and Saving Models

The GNA plugin supports loading and saving of the GNA-optimized model
(non-IR) via the `-rg` and `-wg` flags.  Thereby, it is possible to avoid
the cost of full model quantization at run time. The GNA plugin also
supports export of firmware-compatible embedded model images for the
Intel^&reg; Speech Enabling Developer Kit and Amazon Alexa Premium
Far-Field Voice Development Kit via the -we flag (save only).

In addition to performing inference directly from a GNA model file, these options make it possible to:
- Convert from IR format to GNA format model file (`-m`, `-wg`)
- Convert from IR format to embedded format model file (`-m`, `-we`)
- Convert from GNA format to embedded format model file (`-rg`, `-we`)

## Use of Sample in Kaldi Speech Recognition Pipeline

The Wall Street Journal DNN model used in this example was prepared
using the Kaldi s5 recipe and the Kaldi Nnet (nnet1) framework.  It is
possible to recognize speech by substituting the speech_sample for
Kaldi's nnet-forward command.  Since the speech_sample does not yet 
use pipes, it is necessary to use temporary files for speaker-
transformed feature vectors and scores when running the Kaldi speech
recognition pipeline.  The following operations assume that feature
extraction was already performed according to the `s5` recipe and that
the working directory within the Kaldi source tree is `egs/wsj/s5`.
1. Prepare a speaker-transformed feature set given the feature transform specified in `final.feature_transform` and the feature files specified in `feats.scp`:

```
nnet-forward --use-gpu=no final.feature_transform "ark,s,cs:copy-feats scp:feats.scp ark:- |" ark:feat.ark
```

2. Score the feature set using the `speech_sample`:

```
./speech_sample -d GNA_AUTO -bs 8 -i feat.ark -m wsj_dnn5b_smbr_fp32.xml -o scores.ark
```

3. Run the Kaldi decoder to produce n-best text hypotheses and select most likely text given the WFST (`HCLG.fst`), vocabulary (`words.txt`), and TID/PID mapping (`final.mdl`):

```
latgen-faster-mapped --max-active=7000 --max-mem=50000000 --beam=13.0 --lattice-beam=6.0 --acoustic-scale=0.0833 --allow-partial=true --word-symbol-table=words.txt final.mdl HCLG.fst ark:scores.ark ark:-| lattice-scale --inv-acoustic-scale=13 ark:- ark:- | lattice-best-path --word-symbol-table=words.txt ark:- ark,t:-  > out.txt &
```

4. Run the word error rate tool to check accuracy given the vocabulary (`words.txt`) and reference transcript (`test_filt.txt`):

```
cat out.txt | utils/int2sym.pl -f 2- words.txt | sed s:\<UNK\>::g | compute-wer --text --mode=present ark:test_filt.txt ark,p:-
```

## Links 

- [<b>Main Page</b>](index.html)
- [Use of the Inference Engine](@ref IntegrateIEInApp)
- [<b>Intel's Deep Learning Model Optimizer Developer Guide</b>](https://software.intel.com/en-us/model-optimizer-devguide)
- [Inference Engine Samples](@ref SamplesOverview)
- [<b>Deep Learning Deployment Toolkit Web Page</b>](https://software.intel.com/en-us/computer-vision-sdk)
