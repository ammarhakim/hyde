# Design and Implementation of Hyde

Hyde is a web (browser) based UX for Gkeyll. Although Hyde can be run
on local machines, its key goal is to allow global access to the code
via the WWW.

Hyde depends on the following packages:

- Gkeyll 2.0
- postgkyl
- flask (web-server)
- redis (in-memory DB)
- bokeh (viz; https://bokeh.pydata.org/en/latest/)
- React (front-end client library)

## Data storage policy

All persistent data (except simulation output) is stored in redis. No
data that must persist and is shared shall be stored in an object, but
only in Redis. Python objects will provide methods to CRUD (Create,
Retrive, Update Delete) these data. This indirection allows
maintaining a consistent state for each object.

Simulation output will be stored on-disk in the usual way.

## Communication with Gkeyll

There will be two types of communiation with G2. First, a in-memory G2
instance will be created for each connection. This will allow user to
run code in an interactive way. The second will be a batch mode in
which the input file is run throught G2, potentially in parallel on a
remote machine (different than the server).

The communication will be performed with ZeroMQ and Redis pub/sub.

## User object

The User object stores all information about a user. Guest users are
allowed, although there is no data persistence for guest user. (That
is, once a guest user closes the browser, all server-side data is soon
deleted).

Each User has the following data stored for it (see User.py file for
more information):

- User group (allowing privileged access to features/machies)
- List of user specific templates
- List of running, editing and completed simulations

When a new user is created a directory is made in simroot. All input
files are stored in this directory as sub-directories.

When a user is removed he is marked as 'deleted' but the data is not
immediately removed from the system. This is because immediate removal
may cause problems with already queued and running simulations. A
seperate process (cron job or something else) will periodically cull
users with the 'deleted' tag.

To cull guest users and deleted user, perhaps a thing to do is
schedule cleanup via Redis timed queue, rather than a cron job.

## Simulation object

Simulation object stores all information about a
simulation. Simulations can be of several different types: running,
queued, editing and completed. When a user is editing an input file,
it is in the editing state. When he hits the "run" button it gets into
the queued state. When it starts running it goes into the running
state. Once completed, it is marked completed.

(When a user hits "init" the input file still is pushed into a special
queue and then popped back into the editing state)

This design allows restricting the number of running or queued jobs a
user has.

Data for simulations is stored in redis but the simulation output is
stored in directories (on disk).

Simulation directory is only created with the simulation is either
initialized or run. While editing the input file in editing state the
updated input file is stored in redis.

A SimRunner object manages the redis run queue and running the
simulation. This object will update the simulation state as needed.

In general, only 'editing' simulations can be edited. However, a user
can mark a 'completed' simulation 'editing' to allow an existing
simulation to be edited. This will overwrite the data from the
previous run of the simulation.