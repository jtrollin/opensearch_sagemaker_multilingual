# opensearch_sagemaker_multilingual
Amazon OpenSearch using Amazon SageMaker for multilingual searching
This setups up a Jupyter notebook that will be used to create the demo.

The demo consits of an Amazon OpenSearch cluster and two Amazon SageMaker endpoints.  The endpoints each host a differnt embedding model to show the differnces between the two models.

The OpenSearch cluster is seeded with 30 sentences talking about the season of spring, the 30 sentences talking about a mechanical spring and 30 sentences talking about the verb spring in three different languages (English, French and German) for a total of 90 sentences per language.   This is to show how the mondels handle understanding the meaning of the words you are search for.

### Step 1 - Run cloudformation template

From the AWS console navigate to the CloudFormation dashboard and select **Create stack**, then select **With new resources (standard)**

Under the **Specify template** section, select **Upload a template file**
![Specify template](images/specify_template.png)

Click **Choose file** and select the **setup.yaml** file.

On the next screen name the stack **OpenSearchSageMakerDemo**, if you choose a different name the jupyter notebook code will fail.
![Name stack](images/name_stack.png)

Finish the run stack wizard remembering to check the box on the bottom of the last screen.

Once the Stack is finished, naviate to the **SageMaker** dashboard in the console.

On the left hand menu click on the **Notebook** dropdown menu.
![notebook dashboard](images/notebooks.png)

Click on the **Notebook instances** link.
![notebook instances](images/demo_notebook.png)

Here you should see a notebook that has been created for you.  click on the **Open JupyterLab** link on the right.  This will launch the notebook.

On the right had side of the notebook, you will see a file directory.  Open the file named **OpensearchDemo.ipynb**.
![notebook instances](images/open_notebook.png)

You will be asked to select a runtime for the notebook.  Select **conda_python3**.
![notebook instances](images/choose_runtime.png)

The rest of this tutorial will be run from the notebook you just opened.
