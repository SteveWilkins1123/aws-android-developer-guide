.. Copyright 2010-2016 Amazon.com, Inc. or its affiliates. All Rights Reserved.

   This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0
   International License (the "License"). You may not use this file except in compliance with the
   License. A copy of the License is located at http://creativecommons.org/licenses/by-nc-sa/4.0/.

   This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
   either express or implied. See the License for the specific language governing permissions and
   limitations under the License.

.. highlight:: java


#########################################
Mobile Backend Using Amazon Lambda
#########################################

AWS Lambda is a compute service that runs your code in response to events and automatically manages
the compute resources for you, making it easy to build applications that respond quickly to new
information. The AWS Mobile SDK for Android enables you to call Lambda functions from your Android
mobile apps.

The tutorial below explains how to integrate AWS Lambda with your app.


Project Setup
=============

Prerequisites
-------------

You must complete all of the instructions on the `Set Up the SDK for Android
<http://docs.aws.amazon.com/mobile/sdkforandroid/developerguide/setup.html>`_ page before beginning
this tutorial.


Create a Lambda Function in the AWS Console
-------------------------------------------

For this tutorial, let's use a simple "echo" function that returns the input. Follow the steps
described at `Amazon Lambda Getting Started
<http://docs.aws.amazon.com/lambda/latest/dg/getting-started.html>`_, replacing the function code
with the code below::

 exports.
 handler = function(event, context) {
      console.log("Received event");
      context.succeed("Hello "+ event.firstName + "using " + context.clientContext.deviceManufacturer);
   }


Set IAM Permissions
-------------------

The default IAM role policy grants your users access to Amazon Mobile Analytics and Amazon Cognito
Sync. To use AWS Lambda in your application, you must configure the IAM role policy so that it
allows your application and your users access to AWS Lambda. The following IAM policy allows the
user to perform the actions shown in this tutorial on a given AWS Lambda function identified by its
Amazon Resource Name (ARN). The ARN is found in the Lambda Console by clicking the function name::

   {
      "Statement": [{
         "Effect": "Allow",
         "Action": [
             "lambda:invokefunction"
         ],
         "Resource": [
            "resource-id",
         ]
      }]
   }

To set IAM Permissions for AWS Lambda:

#. Navigate to the :console:`IAM Console <iam/home?region=us-east-1#>` and click :guilabel:`Roles`
   in the left-hand pane.

#. Type your identity pool name into the search box. Two roles will be listed: one for
   unauthenticated users and one for authenticated users.

#. Click the role for unauthenticated users (it will have :code:`unauth` appended to your Identity
   Pool name).

#. Click the :guilabel:`Create Role Policy` button, select :guilabel:`Policy Generator`, and then
   click the :guilabel:`Select` button.

#. Enter a name for your policy and paste in the policy document shown above, replacing the resource
   values with the ARN for your function. You can view the ARN for your function in the AWS Lambda
   Console by clicking the function name. Your ARN will look like this:
   :code:`arn:aws:lambda:us-west-2:account-id:function:invokeFunction`.

#. Click the :guilabel:`Add Statement` button, and then click the :guilabel:`Next Step` button. The
   wizard will show you the configuration that you generated.

#. Click the :guilabel:`Apply Policy` button.

To learn more about IAM policies, see `IAM documentation
<http://docs.aws.amazon.com/IAM/latest/UserGuide/IAM_Introduction.html>`_.


Set Permissions in Your Android Manifest
----------------------------------------

In your :file:`AndroidManifest.xml`, add the following permission

.. code-block:: xml

    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />


Initialize LambdaInvokerFactory
===============================

Pass your initialized Amazon Cognito credentials provider to the :code:`LambdaInvokerFactory` constructor::

  LambdaInvokerFactory factory = new LambdaInvokerFactory(
    myActivity.getApplicationContext(),
    REGION,
    credentialsProvider);


Declare Data Types
==================

Declare the Java classes to hold the data you pass to the Lambda function. The following class
defines a NameInfo class that contains a person's first and last name::

   package com.amazonaws.demo.lambdainvoker;

   /**
    * A simple POJO
    */
    public class NameInfo {
       private String firstName;
       private String lastName;

       public NameInfo() {}

       public NameInfo(String firstName, String lastName) {
           this.firstName = firstName;
           this.lastName = lastName;
       }

       public String getFirstName() {
          return firstName;
       }

       public void setFirstName(String firstName) {
          this.firstName = firstName;
       }

       public String getLastName() {
          return lastName;
       }

       public void setLastName(String lastName) {
           this.lastName = lastName;
       }
   }


Create a Lambda proxy
=====================

Declare an interface containing one method for each Lambda function call. Each method in the
interface must be decorated with the "@LambdaFunction" annotation. The LambdaFunction attribute can
take 3 optional parameters:

- :code:`functionName` allows you to specify the name of the Lambda function to call when the method
  is executed, by default the name of the method is used.

- :code:`logType` is valid only when invocationType is set to "Event". If set, AWS Lambda will
  return the last 4KB of log data produced by your Lambda Function in the x-amz-log-results header.

- :code:`invocationType` specifies how the Lambda function will be invoked. Can be one of the
  following values:

  - Event: calls the Lambda Function asynchronously
  - RequestResponse: calls the Lambda Function synchronously
  - DryRun: allows you to validate access to a Lambda Function without executing it

The following code shows how to create a Lambda proxy::

   package com.amazonaws.demo.lambdainvoker;
   import com.amazonaws.mobileconnectors.lambdainvoker.LambdaFunction;

   /*
    * A holder for lambda functions
    */
   public interface MyInterface {

      /**
       * Invoke lambda function "echo". The function name is the method name
       */
      @LambdaFunction
      String echo(NameInfo nameInfo);

      /**
       * Invoke lambda function "echo". The functionName in the annotation
       * overrides the default which is the method name
       */
      @LambdaFunction(functionName = "echo")
      void noEcho(NameInfo nameInfo);
   }

Invoke the Lambda Function
==========================

.. note:: Do not invoke the Lambda function from the main thread as it results in a network call.

The following code shows how to initialize the Cognito Caching Credentials Provider and invoke a
Lambda function. The value for :code:`IDENTITY_POOL_ID` will be specific to your account. Ensure the
region is the same as the Lambda function you are trying to invoke.

::

    // Create an instance of CognitoCachingCredentialsProvider
    CognitoCachingCredentialsProvider credentialsProvider = new CognitoCachingCredentialsProvider(
         myActivity.getApplicationContext(),
         IDENTITY_POOL_ID,
         Regions.YOUR_REGION);

    // Create a LambdaInvokerFactory, to be used to instantiate the Lambda proxy
    LambdaInvokerFactory factory = new LambdaInvokerFactory(
      myActivity.getApplicationContext(),
      REGION,
      credentialsProvider);

    // Create the Lambda proxy object with default Json data binder.
    // You can provide your own data binder by implementing
    // LambdaDataBinder
    MyInterface myInterface = factory.build(MyInterface.class);

    NameInfo nameInfo = new NameInfo("John", "Doe");

    // The Lambda function invocation results in a network call
    // Make sure it is not called from the main thread
    new AsyncTask<NameInfo, Void, String>() {
        @Override
        protected String doInBackground(NameInfo... params) {
        // invoke "echo" method. In case it fails, it will throw a
        // LambdaFunctionException.
        try {
                return myInterface.echo(params[0]);
         } catch (LambdaFunctionException lfe) {
             Log.e(TAG, "Failed to invoke echo", lfe);
             return null;
          }
     }

    @Override
    protected void onPostExecute(String result) {
        if (result == null) {
            return;
         }

            // Do a toast
            Toast.makeText(MainActivity.this, result, Toast.LENGTH_LONG).show();
        }
    }.execute(nameInfo);

Now whenever the Lambda function is invoked, you should see an application toast with the text
"Hello John using <device>".

For more information on accessing AWS Lambda, see :doc:`lambda`.

.. _Cognito Console: https://console.aws.amazon.com/cognito/home
