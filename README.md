# transform_cognitive_services_workshop

There are several key parts of this workshop.

1. Getting an Azure free subscription and activating it https://azure.microsoft.com/en-us/free/
2. Deploying Azure infrastructure
3. Setting up a local environment with Visual Studio code, python and model training
4. Building Azure function and deploying it.
5. Running it.
6. Do the homework, by switching function to the C# code

Get an Azure access via

Pre-requisites:
* Python v3.8.6(up to 3.9) with PIP, because Azure functions runtime support compatibility.
Azure Function tools
* https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v3%2Cwindows%2Ccsharp%2Cportal%2Cbash%2Ckeda#install-the-azure-functions-core-tools
VSCode is a preferred option.
* Azure CLI tools is an option, you can run a command line from Azure Portal.
https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli

# Creating Azure infrastructure

Go to the Azure Portal and open Bash command line.
Copy/paste script below, make sure to CHANGE the name Microsoft in the first line to correct name of your subscription, to avoid deployment to your production environment.

![image](https://user-images.githubusercontent.com/36765741/204647123-b451e11b-6ad6-45fa-bccf-bc253f024ea1.png)

You might get a delay, because of provider installations with messages like this.
Resource provider 'Microsoft.CognitiveServices' used by this operation is not registered. We are registering for you.

```
subscriptionID=$(az account list --query "[?contains(name,'Microsoft')].[id]" -o tsv)
echo "Test subscription ID is = " $subscriptionID
az account set --subscription $subscriptionID
az account show

location=northeurope
postfix=$RANDOM

export groupName=transform-ai-workshop$postfix
export cognAccName=transformaicogn1$postfix

az group create --name $groupName --location $location

az cognitiveservices account create --name $cognAccName --kind CustomVision.Training --sku F0 --resource-group $groupName --location $location --yes

#----------------------------------------------------------------------------------
# Azure function app with storage account
#----------------------------------------------------------------------------------

accountSku=Standard_LRS
accountName=transformaifunc$postfix

az storage account create --name $accountName --location $location --kind StorageV2 \
--resource-group $groupName --sku $accountSku --access-tier Hot  --https-only true

accountKey=$(az storage account keys list --resource-group $groupName --account-name $accountName --query "[0].value" | tr -d '"')

accountConnString="DefaultEndpointsProtocol=https;AccountName=$accountName;AccountKey=$accountKey;EndpointSuffix=core.windows.net"

applicationName=transformaifunc$postfix
echo "applicationName  = " $applicationName

az functionapp create --name $applicationName --storage-account $accountName --resource-group $groupName --consumption-plan-location $location \
--runtime python --runtime-version 3.8 --functions-version 3 --os-type linux  

az functionapp update --resource-group $groupName --name $applicationName --set dailyMemoryTimeQuota=400000

#----------------------------------------------------------------------------------
# Display secrets
#----------------------------------------------------------------------------------

az cognitiveservices account keys list --name $cognAccName --resource-group $groupName 
az cognitiveservices account show --name $cognAccName --resource-group $groupName -o json --query properties.endpoint

```

Please save the script output to the dock file, so we can use these scripts for 

At this point you should have:
- An endpoint URL looking like this: https://northeurope.api.cognitive.microsoft.com/ 
- 2 keys looking like this: 06a611d19f4f479
  
# 3 Setting up the environment

Make sure you have Python installed(with pip) from https://www.python.org/ by running CMD and executing
```
python -V
pip --version
```
Visual Studio Code - https://code.visualstudio.com/

And please don't forget to save file changes after changes :)

Open terminal in Visual Studio Code from Terminal menu and rund following two commands
```
mkdir globalvision
cd globalvision
```

There is two way to accomplish this workshop, one is via files below, alternative is to add each code section to a new section of Jupiter Notebook.  
  
Create a new file dataset-download.py via terminal window
```
code dataset-download.py
```

Open file and paste following code
```
import os
import urllib.request
import zipfile

# Download the dataset from Github
data_url = "https://github.com/staslebedenko/transform_cognitive_services/raw/main/dataset/download/simpsons-lego.zip"
data_path = "./data-train"
download_path = os.path.join(data_path,"simpsons-lego-dataset.zip")
if not os.path.exists(data_path):
    os.mkdir(data_path);
urllib.request.urlretrieve(data_url, filename=download_path)

# Unzip the dataset
zip_ref = zipfile.ZipFile(download_path, 'r')
zip_ref.extractall(data_path)
zip_ref.close()
print("Data extracted in: {}".format(data_path))

os.remove(download_path)
print("Downloaded file removed: {}".format(download_path))
```

And run file by right clicking on a code and selecting run file or with command line
terminal should be opened in the folder where you created a file
```
python dataset-download.py
```

Wait for download to complete, then install custom vision package and create new file
```
pip install azure-cognitiveservices-vision-customvision
code custom-vision.py
```

Add following imports to file and variables with values from Azure CLI output
Important part is that region northeurope is hardcoded, if you changed Azure CLI for different region, change it in file too

```
from azure.cognitiveservices.vision.customvision.training import CustomVisionTrainingClient
from azure.cognitiveservices.vision.customvision.training.models import ImageFileCreateEntry, ImageFileCreateBatch
from msrest.authentication import ApiKeyCredentials
import time
import os
import urllib.request
import zipfile
import requests

cv_endpoint = "https://northeurope.api.cognitive.microsoft.com"
training_key = "<INSERT KEY 1>"
training_images = "data-train"

credentials = ApiKeyCredentials(in_headers={"Training-key": training_key})
trainer = CustomVisionTrainingClient(endpoint=cv_endpoint,credentials=credentials)
```

Now you are ready to create your first project. The project takes a name and domain as input, the name can be anything. The domain is a different story. You can ask for a list of all possible domains and choose the one closest to what you are trying to accomplish. For instance if you are trying to classify food you pick the domain ???Food??? or ???Landmarks??? for landmarks. Use the code below to show all domains.

```
for domain in trainer.get_domains():
  print(domain.id, "\t", domain.name) 
```

Let???s create a new project with the domain set to ???General Compact???.
And add following string to the file

```
project = trainer.create_project("Lego Simpsons",domain_id="0732100f-1a38-4e49-a514-c9b44c697ab5")
```

Now we need to tag images, so we 

```
image_list = []
directories = os.listdir(training_images)

for tagName in directories:
 	tag = trainer.create_tag(project.id, tagName)
 	images = os.listdir(os.path.join(training_images,tagName))
 	for img in images:
 		with open(os.path.join(training_images,tagName,img), "rb") as image_contents:
 			image_list.append(ImageFileCreateEntry(name=img, contents=image_contents.read(), tag_ids=[tag.id]))  
```

Now we need to create a batches to upload images

```
def chunks(l, n):
 	for i in range(0, len(l), n):
 		yield l[i:i + n]
batchedImages = chunks(image_list, 64)
```

And upload them to Azure
```
for batchOfImages in batchedImages:
    upload_result = trainer.create_images_from_files(project.id, ImageFileCreateBatch(images=batchOfImages))
    if not upload_result.is_batch_successful:
        print("Image batch upload failed.")
        for image in upload_result.images:
            print("Image status: ", image.status)
    else:
        print("Batch uploaded successfully")
print("Done uploading")
```

And finally train your model
```
print ("Training...")
iteration = trainer.train_project(project.id)
while (iteration.status != "Completed"):
    iteration = trainer.get_iteration(project.id, iteration.id)
    print ("Training status: " + iteration.status)
    print ("Waiting 25 seconds...")
    time.sleep(25)
print("Training complete")
```

Now we need to test a ONNX model, so we are adding code to export model

```
platform = "ONNX"
flavor = "ONNX12"
iteration_id = iteration.id 
project_id = project.id
export = trainer.export_iteration(project_id, iteration_id , platform, flavor, raw=False)
```

And monitoring if it is done
```
while (export.status == "Exporting"):
    print ("Waiting 10 seconds...")
    time.sleep(10)
    exports = trainer.get_exports(project.id, iteration_id)
    # Locate the export for this iteration and check its status  
    for e in exports:
        if e.platform == export.platform and e.flavor == export.flavor:
            export = e
            break
    print("Export status is: ", export.status)
print("Export: done")
  
if export.status == "Done":
    # Success, now we can download it
    export_file = requests.get(export.download_uri)
    with open("export.zip", "wb") as file:
        file.write(export_file.content)
        
# Unzip the downloaded export
if not os.path.exists("./model"):
    os.mkdir("./model");
zip_ref = zipfile.ZipFile("export.zip", 'r')
zip_ref.extractall("./model")
zip_ref.close()
print("Data extracted in: ./model")
```

```
python custom-vision.py
```

Create auxilary file to download dataset that our model don't know and validate results
```
code test-dataset-download.py
```

With contents 
```
import os
import urllib.request
import zipfile

# Download the dataset from Github
data_url = "https://github.com/staslebedenko/transform_cognitive_services/raw/main/dataset/download/test-images.zip"
data_path = "./data-test"
download_path = os.path.join(data_path,"test-images.zip")
if not os.path.exists(data_path):
    os.mkdir(data_path);
urllib.request.urlretrieve(data_url, filename=download_path)

# Unzip the dataset
zip_ref = zipfile.ZipFile(download_path, 'r')
zip_ref.extractall(data_path)
zip_ref.close()
print("Data extracted in: {}".format(data_path))

os.remove(download_path)
print("Downloaded file removed: {}".format(download_path))
```

We added code in chunks with explanations, now we will run it with

```
python test-dataset-download.py
```

And now we need to test if our model works. So we creating a new file

```
code validate-model.py
```

install a few pip packages
  
```
pip install onnxruntime
pip install numpy
pip install Pillow
```
  
With code
```
import onnxruntime as nxrun
import numpy as np
import PIL
from PIL import Image
import os

training_images = "./data-test"
model_path = "./model/model.onnx"

sess = nxrun.InferenceSession(model_path)

testimages = os.listdir(training_images)

for image_filepath in testimages[0:5]:
    image = PIL.Image.open(os.path.join(training_images,image_filepath)).resize([224,224])
    input_array = np.array(image, dtype=np.float32)[np.newaxis, :, :, :]
    input_array = input_array.transpose((0, 3, 1, 2))[:, (2, 1, 0), :, :]

    input_name = sess.get_inputs()[0].name
    outputs = sess.run(None, {input_name: input_array.astype(np.float32)})

    print("Image:", image_filepath)
    print("Label: " + outputs[0][0][0])
    print("Score: " + str(outputs[1][0][outputs[0][0][0]]))
    print("--")
```

Running the file from console, make sure to download the test images with previous file
```
python validate-model.py
```

And now the first part of workshop is concluded.



# 5 Building Azure function.
  
Lets use CMD command line to create a new app with function cli tools.
  
```
func init SimpsonsFunctions --python
cd SimpsonsFunctions 
```  

please install PIP package for azure functions via.
```
pip install azure-functions
```

Be aware, that you should add newly generated local.settings.json to gitignore
  
Create a new function inside your function app
```
func new --name Classify --template "HTTP trigger" --authlevel "anonymous"
```
open the new file(function) in VS Code via command below
```
code Classify/__init__.py 
cd Classify 
```

Replace code inside with 
```
import logging
import onnxruntime as nxrun
import numpy as np
import PIL
from PIL import Image
import requests
from io import BytesIO
import azure.functions as func

def main(req: func.HttpRequest) -> func.HttpResponse:

    ## Get the image and resize it to 244x244 pixels
    url = req.params.get('url')
    response = requests.get(url)
    image = PIL.Image.open(BytesIO(response.content)).resize([224,224])

    ## Load the model and score the image
    model_path = "model/model.onnx"
    sess = nxrun.InferenceSession(model_path)
    input_array = np.array(image, dtype=np.float32)[np.newaxis, :, :, :]
    input_array = input_array.transpose((0, 3, 1, 2))[:, (2, 1, 0), :, :]
    input_name = sess.get_inputs()[0].name
    outputs = sess.run(None, {input_name: input_array.astype(np.float32)})

    ## Find the label with the highest score
    label = outputs[0][0][0]
    score = (outputs[1][0][outputs[0][0][0]]*100)
    
    ## Return and log the response
    response = f"I'm {score:.2f}% sure I see: {label}"
    logging.info(response)
    return func.HttpResponse(response)
```

Copy your model to function from explorer or with cmd
```
cp ../model/ ./model -r
```

Open the requirements.txt. This files contains a list of packages that need to be installed in the Azure Function
```
code requirements.txt
```
And add packages
  
```
onnxruntime
pillow!=8.3.0
requests
```
  
And finally deploy your function app to Azure, use name from Azure CLI script above
  
```
func azure functionapp publish transform-vision-workshop
```


```
https://transformaifunc.azurewebsites.net/api/classify?url=https://github.com/staslebedenko/transform_cognitive_services/raw/main/dataset/test/Bart.jpg
```


Use the url and add parameter, so function can validate the pictures, or you can just google pictures of lego simpsons and pass the direct url.
```
https://st2.depositphotos.com/1009868/6339/i/950/depositphotos_63391419-stock-photo-lego-bart-simpson-minifigure.jpg
https://github.com/staslebedenko/transform_cognitive_services/raw/main/dataset/test/Bart.jpg
https://github.com/staslebedenko/transform_cognitive_services/blob/main/dataset/test/Homer.jpg?raw=true
```

# 5 Homework

If you are C# developer, you might wondering if there is an sample to do it your way. .NET platform have packages to solve the problem

https://onnxruntime.ai/docs/get-started/with-csharp.html

Also you have an approach to use a portal
https://learn.microsoft.com/en-us/azure/cognitive-services/custom-vision-service/get-started-build-detector

and also use direct SDK
