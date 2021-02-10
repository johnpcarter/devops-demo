# devops-demo

##Import into Jenkins##

After running and performing basic setup tasks, click on "New Item".
Choose a name for your project and select "Multi-branch" pipeline to create your project.

Fill out the form as your wish but you will need to add this repository so that the provided Jenkins file will be used.
Under "*Branch Sources*", click "add source" and select "GitHub". Just paste the url of this repo into "Repository URL", namely "https://github.com/johnpcarter/devops-demo".

No credentials are required as this repository is public.

Click validate to to allow Jenkins to check that the URL is good and then scroll to the bottom of the page and click "save".

Jenkins will then pull and validate the Jenkins file.
