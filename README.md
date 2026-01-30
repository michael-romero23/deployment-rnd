# deployment-rnd



# The path at the end tells Docker where the files are
docker build -t nest-app ./nest-project
docker run -p 3000:3000 nest-app

running with name
docker run -d --name my-running-app -p 3000:3000 nest-app


docker build -t nest-app .