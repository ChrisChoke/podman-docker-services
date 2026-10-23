# Immich Quadlets

It's really annoying what things are need to change when you translate
a docker-compose.yml into a quadlet .container file.
So in the case of immich this brings a lot of manual care into the
.container file.

Immich's .env file contains 3 variable which a unusable in quadlets.
And this are: UPLOAD_LOCATION, DB_DATA_LOCATION, IMMICH_VERSION

This variables need to be present at the generator phase of the systemd unit. In short: Systemd doesn't support this!!
So we need to write it plain text into the .container file and need
to care of 3 different files where it is used.

Yeah, you can say: It's not bad, just a setup on the creation phase of the application. But it still annoying in my opinion.
This make the translation everytime a bit more fiddly. So i can understand why people don't want this fiddly steps when they want
deploy another application on there container host.

Maybe i create a template folder below this one and create a short script which create and inject the variables into a final .container
file to ship a bit around this limitation.