# WordPress Slack Notifications
A simple way to send Slack notifications from WordPress.

## Setup
 - Create a [Slack bot for your workspace](https://slack.com/help/articles/115005265703-Create-a-bot-for-your-workspace) (Make sure your bot has the `chat:write` and `chat:write.customize` scopes)
 - Copy the "OAuth Tokens for Your Workspace" value and define it in `wp-config.php` for your site: `define( 'RH_SLACKBOT_TOKEN', '<your token goes here>' );`
 - Upload and activate `class-rh-slack.php` as a plugin or include it from your theme's `functions.php` file
```
$file = get_template_directory() . '/class-rh-slack.php';
if ( file_exists( $file ) ) {
	require_once $file;
}
```

## How do I send a Slack message?

Call the `send_message()` method of the `RH_Slack` class. Pass a message and arguments that get sent to the Slack API. See https://api.slack.com/methods/chat.postMessage

```
$message = 'Howdy :face_with_cowboy_hat: ';
$args = array(
    'username'   => 'Matt Mullenweg',
    'channel'    => 'general',
    'icon_emoji' => ':horse:',
);
RH_Slack::send_message( $message, $args );
```
## What type of Slack notfications happen automatically?

When a post type that supports Slack notifications performs certian actions a Slack notfication will be sent. We hook into the WordPress action `transition_post_status` and compare statuses to determine what type of noticiation to send. This includes when a post is:
 - Published
 - Unpublished
 - Scheduled
 - Unscheduled
 - Trashed
 - Updated

## Examples

### Published
When a post status changes to `publish` from any other status

Links to the published post

![published](/uploads/398ecaea9bfd60928dd2ec16e8167d39/published.jpg)

### Unpublished
When a post status changes to `draft` from `publish`

Links to the edit screen of the post

![unpublished](/uploads/d7594f8b889c2156a6e2586a0197a580/unpublished.jpg)

### Scheduled
When a post status changes to `future` from another status that is not `future`

Links to the scheduled post and includes link to [timeanddate.com](https://www.timeanddate.com) to convert publish time to timezones for Los Angeles, New York, and Paris ([Example](https://www.timeanddate.com/worldclock/converter.html?iso=20221101T033000&p1=137&p2=179&p3=195))

![scheduled](/uploads/ceec91a64780c78249dc3052a0a11763/scheduled.jpg)

### Unscheduled
When a post status changes to anything except `publish` or `future` from `future`

Links to the edit screen of the post

![unscheduled](/uploads/32e65712415b32169e66c5e476af9551/unscheduled.jpg)

### Trashed
When a post status changes to `trash` from `publish`

Links to the trashed posts screen for the post type

![trashed](/uploads/35890dda22108bdcdfa49cf7c2e6c185/trashed.jpg)

### Updated
When a post status changes to `publish` from `publish` and there is non-empty `$_POST` data sent to the server

(Why do we need to check if `$_POST` is not empty? See https://github.com/WordPress/gutenberg/issues/15094#issuecomment-558986406)

Links to the updated post and includes a link to the post revision screen in WP Admin

![updated](/uploads/b09e098b7782d4dc4bcb516c14880012/updated.jpg)

![updated-diff](/uploads/e250858d42af3f712681532064da9ea7/updated-diff.jpg)

## How do I enable post types to support Slack notifications?

1. Add `slack-notifications` to the [supports argument](https://developer.wordpress.org/reference/functions/register_post_type/#supports) when registering your post type
2. Hook in to the filter `rh/slack/notification_post_types` and apend post type key(s) to the array

## Non-Production Environments Safety Check

There is a safety in place to ensure Slack notifications don't get sent in non-production environments. See the line `add_filter( 'rh/slack/send_message/args', array( $this, 'filter_rh_slack_send_message_args' ) );` to disable this. This filter sets the channel where all Slack notifications go to as `#test-alerts` instead of the usual production channels.

## Filters

### `rh/slack/notification_post_types`

### `rh/slack/post_notification/message`

### `rh/slack/post_notification/args`

### `rh/slack/send_message/args`
