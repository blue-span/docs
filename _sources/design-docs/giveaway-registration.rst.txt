=====================================================
Improved Giveaway Registration and Winner Calculation
=====================================================

In the context that we might want to do giveaways perpetually, it might make
sense to invest in improving the giveaway registration process and making winner
calculation fairer.

At the same time, this is also about having a cool/fun project to casually hack
on.

Abstract
--------

- viewers register for a giveaway via a new "*register for current giveaway*"
  button on `bluespan.gg/giveaway`_

- that button starts an OAuth2 flow [#oauth-example]_ which allows the user to
  authenticate themselves via YouTube

- a giveaway winner is selected by:

  - of the people that clicked the *register for current giveaway* button
    (during the current giveaway), sort/weight [#weight]_ by:

    - between the last giveaway and now:

      - number of distinct days a user made a message in the live stream chat

      - number of streams that a user clicked the like button for

- pick random winner, and pair that winner with a random giveaway reward (for
  giveaways that have more than one type of reward)

- winners are announced in chat by *Santa Claus Bot™* and/or *Giveaway Bot™*

.. admonition:: Estimated effort

   1-3 weekends, depending on personal productivity levels

.. [#weight] this means that each user be 0-28x more likely to be selected than
   a user with a weight of 1. A weight of 0 means that they registered, but did
   not like any videos or chat in the stream. A weight of 28 means that over a
   period of 14 days, the user liked every single video, and made at least one
   stream comment every single day (obviously, this user is Lulu). We could
   tweak/rescale this math arbitrarily of course; this is just an example.

.. [#oauth-example] "OAuth2 flow": the same thing that happens when you click *Log In With YouTube* on `streamlabs.com/login`_
.. _`streamlabs.com/login`: https://streamlabs.com/login
.. _`bluespan.gg/giveaway`: https://bluespan.gg/giveaway

How do we track the people that registered?
-------------------------------------------

The bluespan.gg giveaway service will track this state in a local (sqlite_,
probably) database.

The data model is very simple; something like:

.. list-table:: Giveaways
   :header-rows: 1

   * - id
     - start_date
     - end_date
   * - ``ryzen-1``
     - 3 Nov 2019
     - 10 Nov 2019
   * - ``ryzen-2``
     - 11 Nov 2019
     - 24 Nov 2019
   * - `..`
     -
     -

.. list-table:: Registrations
   :header-rows: 1

   * - id
     - giveaway_id
     - username [#channel-displayname]_
   * - ``registration-abc``
     - ``ryzen-1``
     - PigsGoMoo
   * - ``registration-def``
     - ``ryzen-1``
     - billgates
   * - `..`
     -
     -

.. list-table:: Authorizations
   :header-rows: 1

   * - username
     - token
   * - PigsGoMoo
     - ``QQKfzA...``
   * - billgates
     - ``QQIBib...``
   * - `..`
     -

.. _sqlite: https://www.sqlite.org/index.html


How do we know which of Blue Span's streams happened in a giveaway period?
--------------------------------------------------------------------------

Roughly:

.. code-block:: text

   GET https://developers.google.com/youtube/v3/live/docs/liveBroadcasts/list

   "part": "id,snippet"
   "mine": true
   "onBehalfOfContentOwner": "Blue Span"

The `liveBroadcast`_ resource gives us:

.. code-block:: text

   "id": string
   "actualStartTime": datetime
   "actualEndTime": datetime
   "liveChatId": string

.. _`liveBroadcast`: https://developers.google.com/youtube/v3/live/docs/liveBroadcasts#resource-representation

How do we know which users chatted during a particular stream?
--------------------------------------------------------------

Given a ``liveChatId``, pretty much dump the entire chat log:

.. code-block:: text

   GET https://www.googleapis.com/youtube/v3/liveChat/messages

   "part": "authorDetails"
   "liveChatId": string

The `liveChatMessages`_ resource gives us, for each message:

.. code-block:: text

  "channelId": string
  "displayName": string

.. [#channel-displayname] This is why it is partially wrong to ask for a
   "YouTube username". When you log in to YouTube, you are actually a
   "channel". A more strictly precise word for "YouTube username" is "channel
   displayName".

.. _`liveChatMessages`: https://developers.google.com/youtube/v3/live/docs/liveChatMessages#resource-representation

How do we know which of Blue Span's streams a user liked?
---------------------------------------------------------

Also not hard.

We already have OAuth tokens for each registered user with the
``https://www.googleapis.com/auth/youtube.readonly`` scope. This gives us the
ability to make API calls like:

.. code-block::

   GET https://www.googleapis.com/youtube/v3/videos

   "part": "snippet"
   "myRating": "like"
   "id": "broadcastId1,broadcastId2,..,broadcastId14"

…on behalf of that user, where each `broadcastId1,broadcastId2,..,broadcastId14`
represents a broadcast returned from our ``liveBroadcasts/list`` call earlier.
