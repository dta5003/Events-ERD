Events ERD
=======

### The Exercise

> We have an application that we want to improve by providing the users with a way to create
> events. Users should be able to create an event, invite people, and receive an RSVP.
> Provide an ERD showing the new database tables/fields necessary to implement this
> feature. Then, briefly describe how your model handles recurring events including the
> positives and negatives of your approach.

### My Implementation

#### Goals and Assumptions

I designed this ERD with the following priorities in mind: performance, feature-richness, and simplicity (in that order).

I did not attempt to come up with every possible field for each table, but focused on the fields defining the relationships between the tables and the fields relevant to defining recurring events.

My goals for the system:
* Speed.  The data is structured so reads are fast and require little manipulation.  Any heavy work to be done on write.
* Allow for recurring events to be defined by 'every x days' type rules and 'every 10th day of the month in March that is in the 2nd week and also a Tuesday' type rules.
* Allow for editing of recurring events to change the details about the event (name/location/description) or list of invitees, and to have that edit be able to be applied to every future event or a single instance of the event.

Some assumptions I made:
* Users are able to invite other users. (Invites could be extended to non-users with no major change.  Including a more robust per-user address book would be more complex.)
* All users have permission to create events and invite other users to their events.  A tiered permission system (admin, moderator, user) was not considered.
* Time zone issues aren't considered.  Time fields use seconds since Unix epoch and duration fields use seconds to keep the math simple.  Julian/Mayan/Time Cube calendars not supported.

#### Defining Events

Single events are straightforward.  Insert into Single_Event and Single_Invite and wait for the RSVPs to roll in.

For recurring events, the main points are the separation of single/recurring events and a recurring event being able to have multiple event schedules.

Insert the recurring event and invite data like a single event, then define the recurring condition with one or more rows in Event_Schedule.  Want to have a meeting every 3 days in the morning and also every Friday at lunch?  We can do that.  Put in one row with repeat_start_time at 10AM and repeat_interval of 3 days.  Put in another with repeat_start_time at noon and repeat_weekday set to Friday.  Both refer back to the same recurring event.

Note:  not evident from the ERD - the Event_Schedule table has a constraint on it restricting each row to having either a repeat_interval, or a repeat_month/day/week/weekday combo.  An event may be scheduled with both types of rules simultaneously, but it would require separate rows.

Also, the repeat_month/day/week/weekday fields are bitmasks - you can repeat every 1st/11th/21st, Monday/Wednesday/Friday, June/July/August in one row.

#### Editing Events

Editing the event info or invite list for a single event is, again, straightforward.  Just go ahead and edit.

For recurring events, we have a choice when editing - do we modify the entire set of recurring events or one instance?  In the case of editing the whole set, it is just like a single event.

Editing one instance of a recurring event is where the magic happens.  We create a single event/invite list with the edited data.  Then we edit the event schedule to snip out that instance using repeat_start_time and repeat_end_time.

For example, say we want to invite Bob to just one of our weekly pizza outings.  We create a single event/invite list for that instance of the recurring event and add Bob.  We then modify the Event_Schedule repeat_end_time to stop just before Bob's lucky day and insert new rows with repeat_start_time set to the next weekly outing.

All this logic is done at the time of the edit and preserves the speedy reading of the data.