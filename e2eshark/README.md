 # e2eshark framework-to-iree-to-inference tests

 This is work in progress.

 This test suite enables developers to add small to large (full model)
 end-2-end tests that compares result of running a model in Framework (e.g. Pytorch) 
 against running the commpiled model on a target backend using IREE. If the results 
 are within a tolerable limit, then test passes else test fails. If there are 
 compilation errors, then test will report the stage at which compilation failed.
 
 Test is organized starting with framework: pytorch, tensorflow, onnx
 For each framework category, multiple modes are tested. 

 - pytroch : starting model is a pytorch model
 - tensorflow : stating model is a tensorflow model (no test at present, planned for later)
 - onnx : starting model is an onnx model (work in progress in setting this up)
 
 Following up-to three modes are supported based upon what is possible for a framework:

 - direct: Framework -> Torch MLIR -> Compiled artefact -> Run target backend
 - onnx: Framework -> ONNX -> Import as torch onnx in Torch MLIR -> Torch MLIR -> Compiled artefact -> Run target backend
 - ort: Framework -> ONNX -> Load in IREE ONNX Runtime EP -> Compiled artefact -> Run target backend

 If Framework is onnx, then mode 'direct' will mean same as 'onnx'. For onnx/operators, onnx/combinations the onnx model should be created using ONNX API. For onnx/models, a prebuilt onnx should be checked in as a zip file.
 
 The target backend can be any IREE supported backend: llvm-cpu, amd-aie etc.

## Contents
 The contents are as below. Substitute 'Framework' by one of: pytorch, tensoflow, onnx
 - requirements.txt : 'pip install -r requirements.txt' to install needed additional packages not in your venv or conda
 - run.py : Run 'python run.py --help' to learn about the script. This is the script to run a specific
            test, all tests in a framework, all frameworks as per choice of a user
 - Framework/test_template: This has a template for creating a new test. Copy this to a new test directory
          and modify the TEMPLATE_* identifiers to create a new test
 - Framework/operators: This has operator level test. example: pytorch/operators/conv2d
 - Framework/combinations: This has small test testing combination of operators such as 
            pytorch/combinations/mlp testing a multi layer perceptron which has torch.linear 
            followed by torch.relu and repeated a few times
 - Framework/models: This has full model test. Since this is full model test, you may need necesary 
            permsisions to download a model such as for llama2 you will need hugging face token. You
            should run 'huggingface-cli login' and enter the HF token before launching the test.
            Also set environment variable HF_HOME to a location with enough space. in bash, 
            export HF_HOME="your path"/HF_HOME

 - tools/onnxutil.py : Allows examining an ONNX protobuf file
 - tools/stubs/onnxmodel.py : This is concatenated to 'model.py' in test directory to form a 
                              runmodel.py for onnx input model
 - tools/stubs/pytorchmodel.py : This is concatenated to 'model.py' in test directory to form a 
                                 runmodel.py for pytorch input model
 
 The logs are created as .log files in the test run sub directory. Examine the logs to find and fix 
 cause of any failure.

## Examples

### Running tests

Run the tests in models, operators, combinations of the default framework (i.e. pytorch),
use framework to onnx to torch MLIR path- and run up to hardware inference on default llvm-cpu
hardware target
```
python ./run.py --upto inference -j 4 -c ../../torch-mlir/build -g models combinations operators -i ../../iree-build --mode onnx
```

Print number of occureneces (frequency) of each operator kind in an ONNX file and print 
total count of operator instances

 ```
python ./tools/onnxutil.py onnx/models/resnet50_vaiq_int8/model.onnx -f
```

Run given test onnx/combinations/constant_constantofshape upto inference on target backend
```
python ./run.py -c ../../torch-mlir/build --frameworks onnx --upto inference -i ../../iree-build/ --tests onnx/combinations/constant_constantofshape
```

### Adding new tests

Let us say you wanted to add a new test to the framework "pytorch" to test maxpool  i.e. start with
pytorch model of maxpool, use run of the model as reference gold output and compare IREE compiled
output on target backend. 
First google pytorch maxpool and read about the corresponding behavior of
maxpool operator in pytorch. For this example:
https://pytorch.org/docs/stable/generated/torch.nn.MaxPool2d.html provides description. 
Now take following steps:
1. Use an appropriate suffix to describe what you are intending to test say maxpool_2d_large
2. mkdir -p pytorch/operators/maxpool_2d_large
3. cd pytorch/operators/maxpool_2d_large
4. cp -pr pytorch/operators/conv2d/model.py .
5. modify model.py
6. You can run 'python ./model.py' to see input and output values printed to test 
7. 
8. once your model.py is ready, you can go to root of the e2eshark test directory and run test as below 
   ```
   python ./run.py --upto torch-mlir -c "your torch mlir build dir" --tests pytorch/operators/maxpool_2d_large --mode direct
   ```
   Rerun above with --mode onnx if you want ONNX to get generated from pytorch and tested in addition.

   If you want to test up to inference, then provide your iree build in addition as -i option and run as

   ```
   python ./run.py --upto inference -c "your torch mlir build dir" -i "your iree build dir" --tests pytorch/operators/maxpool_2d_large --mode direct
   ```

   Rerun above with --mode onnx if you want ONNX to get generated from pytorch and tested in addition.

   You will see test-run/pytorch/operators/maxpool_2d_large and logs created. Examine that to see errors. 
   Iterate over fixing your model.py and examing logs till you get it right. Given state of tools, tests may fail.
   If test fails because tool has bug, you are ready to add test. If test fails because of issue in model.py,
   you need to fix that before you can add test.
   
   Once you are satisfied, git add pytorch/operators/maxpool_2d_large/model.py, commit and publish it