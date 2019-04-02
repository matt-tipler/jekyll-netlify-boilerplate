---
layout: post
title: Cloud Firestore Backups
author: jacob
date: '2019-04-02 13:04:50'
categories: development
image: /assets/images/desk-and-mac.jpg
---
<!-- cSpell:ignore firestore coldline devs gcloud datastore pubsub -->
# 1. Cloud Firestore Backups

- [1. Cloud Firestore Backups](#1-cloud-firestore-backups)
  - [1.1. The Context](#11-the-context)
  - [1.2. Setting up the Storage Bucket](#12-setting-up-the-storage-bucket)
  - [1.3. Installing Cloud SDK](#13-installing-cloud-sdk)
  - [1.4. The Juicy Part](#14-the-juicy-part)
  - [1.5. Restoring a backup](#15-restoring-a-backup)
- [2. The Jargon](#2-the-jargon)
- [3. GOTCHA!](#3-gotcha)
  - [3.1. Digging up a buried Collection](#31-digging-up-a-buried-collection)
  - [3.2. Pull that trigger!](#32-pull-that-trigger)
- [4. A U T O M A T I O N !](#4-a-u-t-o-m-a-t-i-o-n)



## 1.1. The Context
We are working on an internal PWA that makes use of Cloud Firestore to store data. We have a test and production database. We use the test DB for our development build, and for testing datamodel transformations. At the time of writing, the app hasn't been deployed to production and we've been waiting for firestore to provide a backup solution.

This, for the most case is fine until we (I) messed up a script when working on test, and accidentally deleted a field in all documents within a core collection. Since this was test, and we were modifying the data structure in prep for prod, we had to export the data from the old SQL database again and pair it up with data from the production database, so the original data could be re-generated. Of course the production database was untouched, just an inconvenience when multiple devs are working on various projects that all use the same test database.

After this incident, we looked at a way to automate backups from the Cloud Firestore, only to find the documentation is very cryptic and does not give proper examples. I am referring to the mysterious `--kind` and `--namespaces` flags when exporting data. (Go to [The Jargon](#2-the-jargon) for more info.)

## 1.2. Setting up the Storage Bucket
For this, I'm assuming that you already have a project set up. Go to the [storage page](https://console.cloud.google.com/storage/browser), and click "create bucket".

![Creating a storage bucket](https://i.imgur.com/kuT8rdA.png)

These are the settings we go for. Coldline is a slower but cheaper storage solution when storing large amounts of data with very little interaction, making it perfect for backups. It costs more than usual to download the data back form the server, so if you think your team may make a blunder on a frequent basis, it may be worth going for a different storage type.

Once this is created, you're ready to export your database to this empty bucket!

## 1.3. Installing Cloud SDK

Go to the [Google SDK Docs](https://cloud.google.com/sdk/) and download the installer. Install it, and you should see a console pop up, that asks if you want to log in. Enter `Y` and a new tab will open up in your browser so that it can access your Google Account.

Next, we select the project we want to interact with:
![Select the project](https://i.imgur.com/Rq7Ojfy.png)

Side-note: If you have already selected a project, you can use the command `gcloud projects list` to list all projects by ID, and then you use `gcloud config set project {PROJECT_ID}` to switch.

## 1.4. The Juicy Part
I have created a dummy Cloud Firestore to play around with. In it, are 2 collection, each with 2 documents, all of which have a `modified` field set to `false`.

![A Cloud Firestore database](https://i.imgur.com/PMcThXf.png)

In order to backup this Firestore, we need to grab the name for the Storage Bucket that we created earlier. In my case, it is `jacob-firestore-backup`.

To backup this database, we need to run the command:

    gcloud datastore export gs://{BUCKET}

So, in my case, I would run `gcloud datastore export gs://jacob-firestore-backup`:

![Backup Command](https://i.imgur.com/Jxo85wA.png)

The `outputUrlPrefix` is the path of the newly-created backup in your Bucket. It can now be found in your bucket (may need to refresh)

![Storage Bucket after exporting](https://i.imgur.com/sMlgzJx.png)


## 1.5. Restoring a backup
For the sake of demonstration, I have modified one of the documents from each collection (changed `modified` from `false` to `true`):
![collection-1 after backup, before restore](https://i.imgur.com/fX8p2tl.png)
![collection-2 after backup, before restore](https://i.imgur.com/0Y5HzQk.png)

The command to restore a backup is:

    gcloud datastore import gs://{BUCKET}/{backup_time}/{backup_time}.overall_export_metadata

So in my case, I would run:

    gcloud datastore import gs://jacob-firestore-backup/2019-03-26T10:36:30_23730/2019-03-26T10:36:30_23730.overall_export_metadata

![Restored](https://i.imgur.com/8rSQ0B5.png)

And now if I look in the Firestore, it looks like this (back to how it was before):
![collection-1 after restore](https://i.imgur.com/TFdR76v.png)
![collection-2 after restore](https://i.imgur.com/oeHDAtB.png)

# 2. The Jargon

The [Google Cloud documentation](https://cloud.google.com/datastore/docs/export-import-entities#exporting_entities) helpfully shows that there is a `--namespaces="(default)"` and `--kind` argument, without *any* mention of what they are for. After doing some digging into the deep depths of the interwebs, I found out that [kinds](https://cloud.google.com/datastore/docs/concepts/entities#datastore-datastore-key-with-parent-nodejs) is the `collection`. From what I have figured out, you can only restore a specific, top level collection. So, to backup, use the following command:

    gcloud datastore export --kinds="collection-1" gs://jacob-firestore-backup

Restoring this backup will of course only restore that collection.

A `namespace` is a term that has been brought over from the old [datastore](https://cloud.google.com/datastore/docs/firestore-or-datastore#in_datastore_mode). For a bog-standard firestore, you do not need to worry about this flag, and can safely pass `undefined` or `(default)` when required. `Kinds` are still somewhat of a mystery to me, especially when it comes to restoring from a deeply rooted collection within a firestore.

# 3. GOTCHA!
## 3.1. Digging up a buried Collection
From what I have tested, you can backup any depth within the database structure (e.g. `gcloud datastore export --kinds="collection-1/ENBGcHSxcqtQEq8vwiSd" gs://jacob-firestore-backup`), but you cannot ***restore that backup!!!***

You just get

    ERROR: (gcloud.datastore.import) Missing input file /jacob-firestore-backup/2019-03-26T15:23:50_77106/all_namespaces/kind_collection-1/ENBGcHSxcqtQEq8vwiSd/all_namespaces_kind_collection-1/output-0

as the response.

## 3.2. Pull that trigger!
Another important note is that when importing data, it will not run the Firestore triggers.

The only solution we can think of is to take a complete backup. That way, when you restore, any sub-collections that rely on root collections are all in sync.

This highlights that whilst restoring the data to firestore appears trivial, the design of cloud based systems with eventually consistent data via database triggers, pubsub queues and disconnect clients makes restoring back to a known point actually very difficult.

A partial solution may be to programmatically deny write access to the Firestore in order to halt database activity, and handle permission errors in the client. We had a lengthy discussion around solving this issue, and given that our PWA works offline, means that bad data could get restored unintentionally via user actions done offline.

So in conclusion, restoring a backup on the production database for us will be a major event, where we would probably restore the backup to a new project and compare the two, using information that suggests it needs a restore in the first place.

# 4. A U T O M A T I O N !
For scheduling a backup, we used [Tania's blog post](https://medium.com/@nedavniat/how-to-perform-and-schedule-firestore-backups-with-google-cloud-platform-and-nodejs-be44bbcd64ae). Essentially, you create a pubsub topic, which when triggered via a cron job, will generate a backup.
