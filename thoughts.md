# Thoughts from exercise

## Learnings - Azure Container Apps

- The Container Apps subnet requires a minimum `/21` prefix (2048 addresses). That's quite a big range. Am I sure this is required?
- The PowerShell objects are hard to work with - they don't seem to contain the latest changes/features
- The AZ CLI isn't great for ACA - it is down as being `preview`. Which surprises me after 2+ years, but the quality matches this - I've had quite a lot of problems with the documentation being poor, or a number of instances where it just doesn't work. Examples are the `command` and `args` parameters, which don't allow the proper values to be passed through. I had to use a YAML file as a workaround.
- Although I can get the ASA apps to scale to zero - if I want to use a private DNS Zone, this isn't free. Although would be cheap enough for enterprise purposes.
- It take a while to run, almost 10 minutes for the relatively simple setup. Main points are ACA environment, the private DNS and the private DNS endpoints.
- Container Apps Environments take an age to delete. They seem to get `scheduled`
- Azure creates additional Resource Groups to support it - that you can't administer
- Struggling to get the `azure containerapps exec` command to run commands on the main app container. I had to just interactively exec into the jumper, and run the commands from there.
- I love being able to set env vars on the jumper, so that you can just do `curl $APP_URL`
- Having the jumper die after 10 minutes seems to work quite well.

## Learnings - Playbooks

- I think these could be useful
- Very Python focused
- DevContainers are therefore a requirement to keep things tidy
- I like being able to create Markdown automatically using `.git/hooks/pre-commit` hooks
- I like being able to remove the playbook output using the `.git/hooks/pre-commit` hooks
- Couldn't find a nice way to stop the playbook running all the way through, so had to hack an `exit 1`
- Playbook code sections were a pain, until I realised I needed to configure it to the use the `bash` kernel, but the `shellscript` language in the code blocks.
- Playbooks needed a bit of work so that they work on both Ubuntu and MacOS (still haven't tested it on Ubuntu, WSL or Windows to be fair)

## To Do

- Custom main app, that is a REST API. Returning HTML when I make a request is a little naff.
- Try out the GPU support (West US 3, Australia East, and Sweden Central)
- Get the `exec` working to run commands (might I have to use the REST API directly?)
