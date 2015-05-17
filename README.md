# Full Stack Web Developer Nanodegree - P3: Item Catalog

This is the project 3 implementation for the Full Stack Web Developer
Nanodegree, implementing a web site based on Flask microframework,
SQLAlchemy as ORM layer, and using OAuth for authentication.

## Prerequisites

To use this program you need the
[Google App Engine Python SDK](https://cloud.google.com/appengine/downloads#Google_App_Engine_SDK_for_Python)
and `Python 2.7` - it comes often with your UNIX or OSX system; download for
Windows at [Python.org](https://www.python.org/downloads/windows/).

If you want to deploy the application on the real cloud App Engine, update the
value of `application` in `app.yaml` to the app ID you have registered in the
App Engine admin console and would like to use to host your instance of this
sample.
For anything but testing on the API Console you will also have to update the
values at the top of `settings.py` to reflect the respective client IDs you have
registered in the [Developer Console](https://console.developers.google.com/)

## Running the application

Use the GUI of App Engine SDK to add the application, and click on `Run`.
Alternatively, with the command line, the application can be started with
`dev_appserver.py appfolder`.

## Data Model

In the existing code, `Conference` is already a child entity of `Profile`.
Following this logic I have modelled `Speaker` as child entity of `Profile`
as well.

`Session` is modelled as a child entity of `Conference`.
In a small deviation from existing code I have chosing to use a KeyProperty
for storing the speakers of a session (instead of urlsafe-versions of the key).

For the wishlist I tried to follow the logic of registration. The user's profile
contains a list of sessions which they are interested in.

With the current data model all conference-related data (except participants)
are in one single entity group, which limits the need for cross-group transactions.
At the moment I am using transactions in the update functions (as they are partial
and only change provided data) and in the session wishlist (as this requires
appending to the wishlist field in the profile).

## RESTful URLs

I tried to implement RESTful URLs as much as possible. The URIs describe the
resources being affected, the verb descibes the action being done on them.
GET is used for queries, POST for insertions, PUT for modifications, and DELETE
for deletions.
One exception is the function `querySessions` which uses POST due to the potentially
larger data which has to be transmitted from the client to the application.

The URIs of speakers contains the conference as a part.

## Task 1: Add Sessions to a Conference

Speakers are modeled as a separate entity, they are modeled as child entities of
the profile which is parent of the conference.
Sessions are modeled as child entities of the conference.
I modeled two forms for `Speaker` and `Session`: one for outgoing information,
and one for inbound information. This way, when creating a session, it is clear
that a speaker_key is required; while when returning a session all the speaker
information is returned.
The type of session is modeled as an Enum.

In the application I use the Session's ID for referencing the session.
While `urlsafe`-keys look opaque, they actually contain some information, e.g.
the existing application the email address of the conference organizer. So it is
presumably better to use the ID instead of the urlsafe-key, I was however too lazy
to make this consistently everywhere.

The following endpoint methods were implemented:
 - `createSpeaker` - POST method, `speaker` as path
 - `updateSpeaker` - PUT method, `speaker/{websafeSpeakerKey}` as path
 - `getSpeaker` - GET method, `speaker/{websafeSpeakerKey}` as path
 - `getSpeakersCreated` - GET method, `speaker/created` as path
 - `createSession` - POST method, `conference/{websafeConferenceKey}/session` as path
 - `getConferenceSessions` - GET method, `conference/{websafeConferenceKey}/session` as path
 - `updateSession` - PUT method, `conference/{websafeConerenceKey}/session/{sessionId}` as path
 - `getSession` - GET method, `conference/{websafeConferenceKey}/session/{sessionId}` as path
 - `getSessionsBySpeaker` - GET method, `speaker/{websafeSpeakerKey}/session` as path
 - `getConferenceSessionsByHighlight` - GET method, `conference/{websafeConferenceKey}/byHighlight/{highlight}` as path
 - `getConferenceSessionsBySpeaker` - GET method, `conference/{websafeConferenceKey}/bySpeaker/{websafeSpeakerKey}` as path
 - `getConferenceSessionsByType` - GET method, `conference/{websafeConferenceKey}/byType/{typeOfSession}` as path
 - `querySessions` - POST method, `conference/{websafeConferenceKey}/session/query` as path

The application is written in python using the Flask microframework.
For authentication I am using Sign-In with Google which is based on OAuth.
A small decorator helps me to signify the functions which cannot be executed
without proper authentication.

Every user with a Google account can use the application, add new elements,
and update/delete existing ones. This would have to be restricted in a real-world
example, but for simplicity I kept it like this.

## Task 2: Add Sessions to User Wishlist

The user's wishlist is stored in an attribute of the profile. This follows the way
conference registration is implemented and also makes fully sense.
One user will hardly add many sessions at the same time, causing little issues of
concurrency.
However, many users might add the same session to their wishlist at the same time.
Would the session "know" which users are interested in it, write operations would
often have to be restarted due to consistency violations and transaction aborts.

## Task 3: Work on indexes and queries

### Create Indexes

Additional indexes are defined in `index.yaml`.

### Come up with 2 additional queries

The additional queries I implemented are `getConferenceSessionsBySpeaker`,
`getConferenceSessionsByHighlight`, and `querySessions`.
This allows a user to get sessions by some speaker in a given conference,
and it allows to get sessions in a conference with a specific highlight.
In addition the function `querySessions`, which is modeled after the
equivalent function for conferences, allows a wide range of custom queries
to be performed.

### Solve the following query related problem

Due to the usage of indexes, Data Store cannot handle a query where two
inequality filters are defined for two different properties.

There are two basic families of solution:
 - avoid the inequality operator on all but one property, using
   e.g. computed properties or IN-operator.
 - perform some additional filtering in python in a loop

While the first option is normally more performant and less costly
(since python execution time is charged), the second option is sometimes
the only way to solve the problem.
E.g. if my typeOfSession would have been a freetext field it would be the
only option.

## Task 4: Add a Task

After a session is added a task is queued and will be executed (possibly
with delay) in the background.
The Task loops through the speakers of the new session and tries to find
other sessions where the speaker is involved. If it finds more than one
session the speaker and all their sessions are put into memcache as
featured.

A new endpoint `getFeaturedSpeaker` has been added which reads the value
from memcache and returns it to the caller.

Only one speaker is featured, and no sorting is done on who this is.
If the memcache entry expires the featured speaker will no longer be
displayed.

## Copyright and license

Some of the application code have been copied from documentation or from
the existing application.
My contributions are in the public domain.
