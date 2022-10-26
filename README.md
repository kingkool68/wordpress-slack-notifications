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

![image](https://user-images.githubusercontent.com/867430/196851051-d193861b-2971-4a44-b046-eafbaae2a59c.png)

### Unpublished
When a post status changes to `draft` from `publish`

Links to the edit screen of the post

![image](https://user-images.githubusercontent.com/867430/196851083-a9655bc7-af2a-4752-8709-f49c2fbbbbc9.png)

### Scheduled
When a post status changes to `future` from another status that is not `future`

Links to the scheduled post and includes link to [timeanddate.com](https://www.timeanddate.com) to convert publish time to timezones for Los Angeles, New York, and Paris ([Example](https://www.timeanddate.com/worldclock/converter.html?iso=20221101T033000&p1=137&p2=179&p3=195))

![image](https://user-images.githubusercontent.com/867430/196851141-9fc9802d-15bb-4a04-88de-7c5c29e454b6.png)

### Unscheduled
When a post status changes to anything except `publish` or `future` from `future`

Links to the edit screen of the post

![image](https://user-images.githubusercontent.com/867430/196851168-e6932de0-7c17-472c-8fbb-a38684dd02ac.png)

### Trashed
When a post status changes to `trash` from `publish`

Links to the trashed posts screen for the post type

![image](https://user-images.githubusercontent.com/867430/196851191-42dfb77b-39a6-4e91-9c31-848d08070377.png)

### Updated
When a post status changes to `publish` from `publish` and there is non-empty `$_POST` data sent to the server

(Why do we need to check if `$_POST` is not empty? See https://github.com/WordPress/gutenberg/issues/15094#issuecomment-558986406)

Links to the updated post and includes a link to the post revision screen in WP Admin

![image](https://user-images.githubusercontent.com/867430/196851210-7178c008-9adc-4e72-b051-8c235d2128b0.png)

![image](https://user-images.githubusercontent.com/867430/196851221-32ce0f43-278f-4a10-86d0-293ef42dcbe9.png)

## How do I enable post types to support Slack notifications?

1. Add `slack-notifications` to the [supports argument](https://developer.wordpress.org/reference/functions/register_post_type/#supports) when registering your post type
2. Hook in to the filter `rh/slack/notification_post_types` and apend post type key(s) to the array

## Non-Production Environments Safety Check

There is a safety in place to ensure Slack notifications don't get sent in non-production environments. See the line `add_filter( 'rh/slack/send_message/args', array( $this, 'filter_rh_slack_send_message_args' ) );` to disable this. This filter sets the channel where all Slack notifications go to as `#test-alerts` instead of the usual production channels.

## Filters

### `rh/slack/notification_post_types`
Modify which post types support Slack notifications.

Example:
```
function ilter_rh_slack_notification_post_types( $post_types = array() ) {
    // Enable Slack notification whenever the book post type content is modified
    $post_types[] = 'book';
    return $post_types;
}
add_filter( 'rh/slack/notification_post_types', 'filter_rh_slack_notification_post_types' );
```

### `rh/slack/post_notification/message`
Modify the Slack message before the notification is sent.

Example:
```
function filter_rh_slack_post_notification_message( $message = '', $new_status = '', $old_status = '', $post ) {
    // Don't send a post updated Slack notification
    if ( $new_status = 'publish' && $old_status = 'publish' ) {
        $message = ''; // Returning an empty string cancels sending the message
    }

    // Change the Slack message when a post is scheduled
    if ( $new_status === 'future' && $old_status !== 'future' ) {
        $the_post_title = get_the_title( $post );
		$the_post_title = html_entity_decode( $the_post_title );
		$the_permalink  = get_permalink( $post );
        $message        = "<$the_permalink|$the_post_title> has been scheduled";
    }

    return $message;
}
add_filter( 'rh/slack/post_notification/message', 'filter_rh_slack_post_notification_message' );
```

### `rh/slack/post_notification/args`
Modify the Slack arguments before a notification is sent.

Example:
```
function filter_rh_slack_post_notification_args( $slack_args = array(), $new_status = '', $old_status = '', $post ) {
    // Change the icon for when a post is trashed
    if ( $new_status === 'trash' && $old_status === 'publish' ) {
        $slack_args['icon_emoji'] = ':poop:';
    }

    // Change the Slack bot username when the post type is Transformers
    if ( $post->post_type === 'transformers' ) {
        $slack_args['username'] = 'Optimus Prime';
    }
    return $slack_args;
}
add_filter( 'rh/slack/post_notification/args', 'filter_rh_slack_post_notification_args' );

### `rh/slack/send_message/args`
Modify the Slack arguments before any Slack message is sent. This comes in handy when you use the `RH_Slack::send_message()` method or you want to ensure Slack messages don't get sent to public channels when you're testing.

Example:
```
function filter_rh_slack_send_message_args( $slack_args = array() ) {
    // If the environemnt is 'development' send all Slack messages to the 'dev-alerts' channel and name the bot 'Dev Bot'
    if ( wp_get_environment_type() === 'development' ) {
        $slack_args['channel']  = 'dev-alerts';
        $slack_args['username'] = 'Dev Bot';
    }
    return $slack_args;
}
add_filter( 'rh/slack/send_message/args', 'filter_rh_slack_send_message_args' );
```
