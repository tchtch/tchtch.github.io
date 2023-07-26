---
layout: post
title: Create an AWS Lambda Function deployable zip package
---

 ![Lambda](/images/Lambda.png)AWS Lambda is a powerful serverless compute service that enables you to run code without managing servers.
                              To deploy your code to AWS Lambda, you need to create a ZIP file containing your Lambda function's code and any required dependencies.

Building the Lambda Function ZIP File
Organize Your Code: Arrange your Lambda function code and any necessary dependencies into a structured directory. Ensure that the main function's entry point is clearly defined.

Zip the Files: Package the code and dependencies into a ZIP file. It's essential to avoid including any unnecessary directories or absolute paths to prevent deployment issues.

Deploying the Lambda Function
AWS Console: Go to the AWS Management Console and navigate to the Lambda service. Click "Create Function," and choose the "Upload a .zip file" option. Upload your ZIP file and configure the function's settings.

AWS CLI: Use the AWS Command Line Interface (CLI) to deploy your function. Use the aws lambda create-function command, specifying the ZIP file location, function details, and required permissions.

Deployment Pipeline: For automated deployments, integrate your Lambda function ZIP file creation and deployment steps into a deployment pipeline, such as AWS CodePipeline or Jenkins.

Testing and Iterating
After deployment, thoroughly test your Lambda function to ensure it behaves as expected. If updates are needed, make changes to your code, create a new ZIP file, and redeploy.

By following these steps, you can successfully create and deploy your AWS Lambda function ZIP file, allowing your serverless application to run seamlessly in the AWS cloud environment. Regularly update and test your function to ensure smooth and efficient deployments for your serverless architecture.

In this post I'll share a Bash Script that automate packaging your python code lambda function : 

[Source in Github](https://github.com/tchtch/Bash_Scripts/blob/main/build_lambda_function_zip_file.sh)   

{% highlight bash %}

#!/usr/bin/env bash
##################################################################################################
## -- GENERAL INFORMATION
## Script  : Create a lambda deployable zip package
## Author  : HA
## Version : v1.0.0
##
## -- SPECIFICATIONS
##      Create a lambda deployable zip package
##
## -- COPYRIGHT
##                        * *    Copyright <2023> <HA>   * *
## Permission is hereby granted, free of charge, to any person obtaining a copy of this script
## and associated documentation files (the "Script"),including without limitation the rights to use, 
## copy, modify, merge, publish, distribute, sublicense.
## 
## THE SCRIPT IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, 
## INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR 
## PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE 
## FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
## ARISING FROM, OUT OF OR IN CONNECTION WITH THE SCRIPT OR THE USE OR OTHER DEALINGS IN THE SCRIPT.
####################################################################################################

# Color variables
red='\033[0;31m'     bg_red='\033[0;41m'
green='\033[0;32m'   bg_green='\033[0;42m'
yellow='\033[0;33m'  bg_yellow='\033[0;43m'
blue='\033[0;34m'    bg_blue='\033[0;44m'
magenta='\033[0;35m' bg_magenta='\033[0;45m'
cyan='\033[0;36m'    bg_cyan='\033[0;46m'
# Special characters
check='✔'            cross='✘'  
# Clear the color
clear='\033[0m'

# Variables
REQUIRED_PYTHON_VERSION="3.9" 			        # Modify this as needed (3.8,2.9,3.10)
PYTHON_VIRTUALENV_DIR="venv" 				    # Modify this as needed
SOURCE_CODE_DIR="python"  	                    # Modify this as needed
LAMBDA_FUNCTION_NAME="lambda_function" 	        # Change this to your function name
LAMBDA_FUNCTION_HANDLER="lambda_handler"  		# Change this to your handler function name
ZIP_PACKAGE_DIR="/tmp" 				            # Modify this as needed
SOFTWARE_LIST="zip Python3 python3-virtualenv"
TEMP_DIR="/${LAMBDA_FUNCTION_NAME}"

# Function to handle Ctrl-C signal
function ctrl_c() {
    echo -e "${red}Ctrl-C: ${magenta}Detected. Aborting...${clear}"
    exit 125
}

## Check if script is run as root   
if [[ $EUID -ne 0 ]]; then
    echo -e "${red}Error: ${magenta}This script must be run as root.${clear}" 
    exit 1
    else
        # Check script running out of working directory
        if [ $(pwd) = "$SOURCE_CODE_DIR" ]; then
            echo -e "${red}Error: ${magenta}Move Script out of the working directory $SOURCE_CODE_DIR, and run it again.${clear}"
            exit 1
        else
            # Check Python version compatibility
            INSTALLED_PYTHON_VERSION=$(python3 -c 'import sys; print(".".join(map(str, sys.version_info[0:2])))')
            if [ "$INSTALLED_PYTHON_VERSION" != "$REQUIRED_PYTHON_VERSION" ]; then
                echo -e "${red}Error: ${magenta} Required Python version $REQUIRED_PYTHON_VERSION not found.${clear}"
                exit 1
            else
            # Check requirements.txt file 
                if [ ! -f "$SOURCE_CODE_DIR/requirements.txt" ]; then
                    echo -e "${yellow}Warning: ${magenta} ${SOURCE_CODE_DIR}/requirements.txt file not found in the current directory.${clear}"
                    sudo touch ${SOURCE_CODE_DIR}/requirements.txt
            fi
        fi
    fi
fi

# Function to determine the package manager used by the OS
function get_package_manager() {
    declare -A osFlavor;
    osFlavor[/etc/debian_version]="sudo apt-get install -y"
    osFlavor[/etc/alpine-release]="sudo apk --update add"
    osFlavor[/etc/centos-release]="sudo yum install -y"
    osFlavor[/etc/fedora-release]="sudo dnf install -y"
    for f in "${!osFlavor[@]}"; do
        if [[ -f "$f" ]]; then
            package_manager="${osFlavor[$f]}"
            break
        fi
    done
    if [ -z "$package_manager" ]; then
        echo -e "${red}Error: ${magenta}Package manager not detected for this OS.${clear}"
        exit 1
    fi
    # Get the list of packages to install (pass as an array argument to the function)
    packages_to_install=("$@")
    if [ ${#packages_to_install[@]} -eq 0 ]; then
        echo -e "${yellow}No packages specified for installation.${clear}"
        return
    fi
    echo -e "${green}-Installing packages...${clear}${check}"
    $package_manager "${packages_to_install[@]}" &>/dev/null;
}

# Function to create a virtual environment, activate it and install requirements 
function create_and_activate_virtualenv() {
    if [ ! -d "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}" ]; then
        echo -e "${green}-Creating virtual environment...${clear}${check}"
        virtualenv -p python3 "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}" &>/dev/null;
    else
       echo -e "${yellow} ${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR} already exists.${clear}"
       read -p "Do you want to continue? (yes/no):" user_input
       user_input_lc=${user_input,,}
       # Check if the user wants to continue or not
       if [ "$user_input_lc" = "yes" ]; then
            echo -e "${green}Continuing with the script..."
            echo -e "-Activating virtual environment...${clear}${check}"
            source "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}/bin/activate"
            if [ -f "${SOURCE_CODE_DIR}/requirements.txt" ]; then
                echo -e "${green}Installing required Python packages...${clear}${check}"
                sudo pip install --no-cache-dir -r ${SOURCE_CODE_DIR}/requirements.txt --ignore-installed --upgrade --target ${TEMP_DIR}
            fi
        else
            echo -e "${magenta}Exiting the script.${clear}"
            exit 0
        fi
    fi
}

# Function to check the existence of the handler function
function check_lambda_function_handler() {
    if ! grep -q "${LAMBDA_FUNCTION_HANDLER}" "${SOURCE_CODE_DIR}/${LAMBDA_FUNCTION_NAME}".py; then
         echo -e "${red}Error: Handler function ${LAMBDA_FUNCTION_HANDLER} not found in ${SOURCE_CODE_DIR}/${LAMBDA_FUNCTION_NAME}.py${clear}"
         exit 1
        else
          echo -e "${green}-Validating Lambda function handler...${clear}${check}"
    fi
}

# Function to check the deployment package size
function build_deployment_package() {
    echo -e "${green}-Packaging Zip file...${clear}${check}" 
    sudo zip -r ${ZIP_PACKAGE_DIR}/${LAMBDA_FUNCTION_NAME}.zip ${ZIP_PACKAGE_DIR} -x '*venv*' -x '*.git*' -x '*boto3*' -x '*botocore*' &>/dev/null;
    echo -e "${magenta}  File Created: ${LAMBDA_FUNCTION_NAME}.zip ${clear}"
    #echo -e "${green}  Checking Zip file size${clear}${check}" 
    package_size=$(du -k ${ZIP_PACKAGE_DIR}/${LAMBDA_FUNCTION_NAME}.zip | cut -f1)
    max_package_size_kb=250000  # Maximum deployment package size in KB (250 MB)
    if [ "$package_size" -gt "$max_package_size_kb" ]; then
        echo -e "${red}Error: Deployment package size exceeds the limit of 250 MB.${clear}"
        exit 1
    else
       echo -e "${magenta}  File Size: $package_size"
       echo -e "  File Location: ${ZIP_PACKAGE_DIR}/${LAMBDA_FUNCTION_NAME}.zip${clear}" 
    fi
}

# Function to clean up build
function clean_up_build() {
    if [ -d "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}" ]; then
        echo -e "${green}Cleanup virtual environment...${clear}"
        read -p "  Do you want to continue? (yes/no): " user_input
        user_input_lc=${user_input,,}
        # Check if the user wants to continue or not
        if [ "$user_input_lc" = "yes" ]; then
           echo -e "${green}Starting cleaning up...${clear}${check}"
           #source "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}"/bin/deactivate
           sudo rm -Rf "${TEMP_DIR}/${PYTHON_VIRTUALENV_DIR}"
           sudo rm -Rf "${TEMP_DIR}"
           echo -e "${green}Cleaning was performed successfuly.${clear}${check}"    
   	else
         echo -e "${magenta}Clean up lambda build venv was not process.${clear}"
         exit 0
        fi
    fi
}

# Function to build lambda function package
function function_package_build() {
    if [ ! -d "${SOURCE_CODE_DIR}" ]; then
        echo -e "${red}ERROR: working directory $SOURCE_CODE_DIR does not exist...${clear}"
        exit 1
    fi
    if [ -d "${TEMP_DIR}" ]; then
        sudo rm -Rf ${TEMP_DIR}
    fi
    echo -e "${green}-Creating build Directory...${clear}${check}"
    sudo mkdir ${TEMP_DIR}
    sudo cp -rf ${SOURCE_CODE_DIR}/* ${TEMP_DIR}/.
    get_package_manager ${SOFTWARE_LIST}
    create_and_activate_virtualenv
    check_lambda_function_handler
    sudo rm -Rf ${TEMP_DIR}/*dist-info
    build_deployment_package
    clean_up_build
}

# main
trap ctrl_c SIGINT
function_package_build

{% endhighlight %}

Have fun coding!
