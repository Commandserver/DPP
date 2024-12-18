\page checking-member-permissions Checking Permissions

Of course most people do just iterate over the roles of a member to check for a permission. But there's a helper method for that: dpp::guild::base_permissions gets a member's permission taking into account the server owner and role permissions.

For total member permissions including channel overwrites use either the dpp::channel::get_user_permissions or dpp::guild::permission_overwrites method. Both do the same under the hood.

They all return a dpp::permission class, which is a wrapper around a permission bitmask containing bits of the dpp::permissions enum.

Demonstration:

```cpp
dpp::channel* c = dpp::find_channel(some_channel_id);
if (c && c->get_user_permissions(member).can(dpp::p_send_messages)) {
	//...
}
```

### Role Hierarchy

The recommended and correct way to compare for roles in the hierarchy is using the comparison operators (`<`, `>`) on the \ref dpp::role::operator<(dpp::role, dpp::role) "dpp::role" objects themselves.
Keep in mind that multiple roles can have the same position number.
As a result, comparing roles by position alone can lead to subtle bugs when checking for role hierarchy.

For example let's say you have a ban command, and want to make sure that any issuer of the command can only ban members lower position than their own highest role.

```cpp
bot.on_interaction_create([](const dpp::interaction_create_t& event) {
    dpp::snowflake target_id = std::get<dpp::snowflake>(event.get_parameter("user"));
    dpp::guild_member target = event.command.get_resolved_member(target_id);

    for (dpp::snowflake issuer_role_id : event.command.member.get_roles()) {
        auto issuer_role = dpp::find_role(issuer_role_id);
        if (issuer_role == nullptr) continue;
        for (dpp::snowflake target_role_id : target.get_roles()) {
            auto target_role = dpp::find_role(target_role_id);
            if (target_role == nullptr) continue;
            if (target_role > issuer_role) {
                event.reply("You can't ban someone whose role is higher than yours!");
                return;
            }
        }
    }
});
```

## Permissions in Interaction Events

### Default Command Permissions

Discord's intended way to manage permissions for commands is through default member permissions. You set them using dpp::slashcommand::set_default_permissions when creating or updating a command to set the default permissions a user must have to use it. However, server administrators can then overwrite these permissions by their own restrictions.

The corresponding code to create a command with default permissions would look something like this:

```cpp
dpp::slashcommand command("ban", "Ban a member", bot.me.id);

command.set_default_permissions(dpp::p_ban_members); // set permissions that are required by default here

command.add_option(dpp::command_option(dpp::co_user, "user", "The user to ban", true));
command.add_option(dpp::command_option(dpp::co_string, "reason", "The reason for banning", true));

bot.global_command_create(command);
```

### Checking Permissions on Your Own

If you want to check permissions on your own, the easiest way to check if a member has certain permissions in interaction events is by using the dpp::interaction::get_resolved_permission function. The resolved list contains associated structures for the command and does not use the cache or require any extra API calls. Note that the permissions in the resolved set are pre-calculated by Discord and taking into account channel overwrites, roles and admin privileges. So no need to loop through roles or stuff like that.

Let's imagine the following scenario:

You have a ban command and want to make sure the issuer has the ban permission.

```cpp
bot.on_interaction_create([](const dpp::interaction_create_t& event) {
	dpp::permission perms = event.command.get_resolved_permission(event.command.usr.id);
	if (!perms.can(dpp::p_ban_members)) {
		event.reply("You don't have the required permissions to ban someone!");
		return;
	}
});
```

\note When using default permissions you don't necessarily need to check the issuing user for any permissions in the interaction event as Discord handles all that for you. But if you'd sleep better...

### From Parameters

The resolved set also contains the permissions of members from command parameters.

For example, let's say you want to prohibit people from banning server admins with your ban command.

Get the user ID from the parameters and pass it to the `get_resolved_permission` method:

```cpp
bot.on_interaction_create([](const dpp::interaction_create_t& event) {
	dpp::snowflake user_id = std::get<dpp::snowflake>(event.get_parameter("user"));
	dpp::permission perms = event.command.get_resolved_permission(user_id);
	if (perms.has(dpp::p_administrator)) {
		event.reply("You can't ban Admins!");
		return;
	}
});
```

### The Bot's Permissions

You also might want to check if the bot itself has the ban permission before processing the command further. You can access the bot's permissions in the dpp::interaction::app_permissions field.

```cpp
bot.on_interaction_create([](const dpp::interaction_create_t& event) {
	if (!event.command.app_permissions.can(dpp::p_ban_members)) {
		event.reply("The bot doesn't have the required permission to ban anyone!");
		return;
	}
});
```
