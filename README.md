# BOTLIB - Telegram C bot framework

Botlib is a C framework to write Telegram bots. It is mainly the sum of two things:

1. An implementation of a subset of the Telegram bot API, wrapped in an event loop that waits for events from the Telegram API and calls our callback in the context of a new thread. The callback that implements the bot has access to various APIs to perform actions in Telegram.
2. A set of higher level wrappers for Sqlite3, JSON, and dynamic strings (SDS library).

## Why this library is written in C?

* First of all, this framework makes writing bots in C a lot more higher level than you could expect. Sqlite3 is exported as a high level API and also exported as a key-value store. Callbacks are called with a structure that already has all the informations about the incoming message, and so forth.
* In the high level languages landscape I had a few bad experiences with libraries changing APIs continuously. A bot is something you write and put online for years: I don't want to babysit code that already works. Bots written with this library will run everywhere as long as you can compile them with `make`. The only dependencies are `libcurl` and `libsqlite3`, which are basically everywhere.
* In the process of writing a few Telegram bots, I found that many requests are quite long living. Think at a bot that transcribes the audio into text, or that uses another external API to fetch information. So multiplexing is not the way to go most of the times: this is why this library uses a thread for each request. And if you use threads like that, you want each thread to be as bare metal as possible, sharing most of the state with the main thread. C is good at that, and the library is implemented so that all the threading issues are transparent for the bot writer.
* Certain bots are quite CPU intensive to run. For instance I wrote a bot that performs analysis on the financial market, and C was a good fit to do Montecaro simulations and things like that.

To give you some feeling about how bots are developed with this framework, see this trivial example, implementing a toy bot:

```c
/* ... standard includes here ... */
#include "botlib.h"

/* For each bot command, private message or group message (but this only works
 * if the bot is set as group admin), this function is called in a new thread,
 * with its private state, sqlite database handle and so forth.
 *
 * For group messages, this function is ONLY called if one of the patterns
 * specified as "triggers" in startBot() matched the message. Otherwise we
 * would spawn threads too often :) */
void handleRequest(sqlite3 *dbhandle, BotRequest *br) {
    char buf[256];
    char *where = br->type == TB_TYPE_PRIVATE ? "privately" : "publicly";
    snprintf(buf, sizeof(buf), "I just %s received: %s", where, br->request);

    /* Let's use our key-value store API on top of Sqlite. If the
     * user in a Telegram group tells "foo is bar" we will set the
     * foo key to bar. Then if somebody write "foo?" and we have an
     * associated key, we reply with what "foo" is. */
    if (br->argc >= 3 && !strcasecmp(br->argv[1],"is")) {
        kvSet(dbhandle,br->argv[0],br->request,0);
        /* Note that in this case we don't use 0 as "from" field, so
         * we are sending a reply to the user, not a general message
         * on the channel. */
        botSendMessage(br->target,"Ok, I'll remember.",br->msg_id);
    }

    int reqlen = strlen(br->request);
    if (br->argc == 1 && reqlen && br->request[reqlen-1] == '?') {
        char *copy = strdup(br->request);
        copy[reqlen-1] = 0;
        printf("Looking for key %s\n", copy);
        sds res = kvGet(dbhandle,copy);
        if (res) {
            botSendMessage(br->target,res,0);
        }
        sdsfree(res);
        free(copy);
    }
}

// This is just called every 1 or 2 seconds. */
void cron(sqlite3 *dbhandle) {
    UNUSED(dbhandle);
    printf("."); fflush(stdout);
}

int main(int argc, char **argv) {
    /* Only group messages matching this list of glob patterns
     * are passed to the callback. */
    static char *triggers[] = {
        "Echo *",
        "Hi!",
        "* is *",
        "*\?",
        "!ls",
        NULL,
    };
    startBot(TB_CREATE_KV_STORE, argc, argv, TB_FLAGS_NONE, handleRequest, cron, triggers);
    return 0; /* Never reached. */
}
```

For the full example, showing other API calls, check the `mybot.c` file in this repository.  See further in this README file for the full API specification.

## Installation

Before developing your bot, it's a good idea to be able to compile and run the example as a Telegram bot.

1. Create your bot using the Telegram [@BotFather](https://t.me/botfather).
2. After obtaining your bot API key, store it into a file called `apikey.txt` inside the bot working directory. Alternatively you can use the `--apikey` command line argument to provide your Telegram API key.
3. Optionally edit `mybot.c` to personalized the bot.
3. Build the bot: you need libcurl and libsqlite installed. Just type `make`.
4. Run with `./mybot`. There is also a debug mode if you run it using the `--debug` option (add --debug multiple times for even more verbose messages). For a more moderate output use `--verbose`. Try `mybot --help` for the full list of command line options.
5. Add the bot to your Telegram channel.
6. **IMPORTANT:** The bot *must* be an administrator of the channel in order to read all the messages that are sent in such channel. Private messages will work regardless.

By default the bot will create an SQLite database in the working directory.
If you want to specify another path for your SQLite db, use the `--dbfile`
command line option.

## Telegram APIs

The library provides a few functions to interact with Telegram. Most of the times you will just use `botSendMessage()` to reply, but here is the full set:

**Sending messages:**

```c
// Send a message. Set reply_to to 0 for a standalone message,
// or to a message ID to reply to that specific message.
int botSendMessage(int64_t target, sds text, int64_t reply_to);

// Same as above, but returns the chat_id and message_id of the sent message.
// Useful if you want to edit the message later.
int botSendMessageAndGetInfo(int64_t target, sds text, int64_t reply_to,
                             int64_t *chat_id, int64_t *message_id);

// Edit a message you previously sent.
int botEditMessageText(int64_t chat_id, int message_id, sds text);

// Send an image file.
int botSendImage(int64_t target, char *filename);
```

**Handling files:** Users can send voice messages, audio files, or documents to your bot. The `BotRequest` structure tells you what was received:

```c
// Check br->file_type to see if a file was attached:
#define TB_FILE_TYPE_NONE 0       // No file
#define TB_FILE_TYPE_VOICE_OGG 1  // Voice message (OGG format)
#define TB_FILE_TYPE_AUDIO 2      // Audio file (MP3, etc.)
#define TB_FILE_TYPE_DOCUMENT 3   // Generic document

// If file_type is not NONE, these fields are populated:
br->file_id    // Telegram's file ID (used to download)
br->file_size  // Size in bytes
br->file_name  // Original filename (audio/document only, may be NULL)
br->file_mime  // MIME type (audio/document only, may be NULL)
```

To download a file, use `botGetFile()`:

```c
if (br->file_type == TB_FILE_TYPE_VOICE_OGG) {
    // Download to a file named "voice.ogg"
    if (botGetFile(br, "voice.ogg")) {
        // File downloaded successfully, do something with it
    }
}
```

**The BotRequest structure:** Your callback receives all the information about the incoming message:

```c
typedef struct BotRequest {
    int type;           // TB_TYPE_PRIVATE, TB_TYPE_GROUP, TB_TYPE_SUPERGROUP, TB_TYPE_CHANNEL
    sds request;        // The message text
    int64_t from;       // User ID of the sender
    sds from_username;  // Username of the sender
    int64_t target;     // Chat ID where to reply
    int64_t msg_id;     // Message ID (use for replies)
    sds *argv;          // Message split into words
    int argc;           // Number of words
    int file_type;      // TB_FILE_TYPE_* (see above)
    sds file_id;        // Telegram file ID
    sds file_name;      // Original filename (if available)
    sds file_mime;      // MIME type (if available)
    int64_t file_size;  // File size in bytes
    int bot_mentioned;  // True if @botname was in the message
    sds *mentions;      // Array of @mentions in the message
    int num_mentions;   // Number of mentions
} BotRequest;
```

**Low-level API:** If you need to call Telegram API methods not wrapped by this library, you can use:

```c
sds makeGETBotRequest(const char *action, int *resptr, char **optlist, int numopt);
```

Where `optlist` is an array of strings alternating parameter names and values, and `numopt` is the number of parameters. The function returns the JSON response as an SDS string.

## Sqlite wrapper API

The library wraps SQLite with a simpler interface. Queries use special placeholders that handle escaping and type conversion automatically:

* `?s` - TEXT field (pass a `char*`)
* `?b` - BLOB field (pass a `char*` followed by `size_t` length)
* `?i` - INTEGER field (pass an `int64_t`)
* `?d` - REAL field (pass a `double`)

**Running queries:**

```c
// INSERT: returns the last inserted row ID, or 0 on error
int64_t id = sqlInsert(dbhandle, "INSERT INTO users VALUES(?i,?s)", user_id, username);

// UPDATE/DELETE: returns 1 on success, 0 on error
int ok = sqlQuery(dbhandle, "UPDATE users SET name=?s WHERE id=?i", new_name, user_id);

// SELECT returning multiple rows:
sqlRow row;
sqlSelect(dbhandle, &row, "SELECT id,name FROM users WHERE active=?i", 1);
while (sqlNextRow(&row)) {
    int64_t id = row.col[0].i;      // Integer column
    const char *name = row.col[1].s; // String column
    printf("User %lld: %s\n", id, name);
}
// Note: sqlNextRow() automatically cleans up when rows are exhausted.
// If you break early, call sqlEnd(&row) to free resources.

// SELECT returning a single row:
sqlRow row;
if (sqlSelectOneRow(dbhandle, &row, "SELECT name FROM users WHERE id=?i", user_id) == SQLITE_ROW) {
    printf("Name: %s\n", row.col[0].s);
}
sqlEnd(&row);

// SELECT returning a single integer:
int64_t count = sqlSelectInt(dbhandle, "SELECT COUNT(*) FROM users");
```

**Key-value store:** The library also provides a simple key-value API on top of SQLite. To use it, include `TB_CREATE_KV_STORE` in your database creation query when calling `startBot()`:

```c
// Set a key (expire=0 means no expiration, otherwise seconds from now)
kvSet(dbhandle, "mykey", "myvalue", 0);        // No expiration
kvSet(dbhandle, "tempkey", "tempvalue", 3600); // Expires in 1 hour

// Set with explicit length (for binary data)
kvSetLen(dbhandle, "binkey", data, datalen, 0);

// Get a key (returns NULL if not found or expired)
sds value = kvGet(dbhandle, "mykey");
if (value) {
    // Use value...
    sdsfree(value);
}

// Delete a key
kvDel(dbhandle, "mykey");
```

## JSON wrapper API

The library includes cJSON for JSON parsing, plus a convenience function `cJSON_Select()` that makes extracting nested values much simpler:

```c
cJSON *json = cJSON_Parse(json_string);

// Select nested fields with a path. The ":s", ":n", etc. suffix
// checks the type and returns NULL if it doesn't match.
cJSON *name = cJSON_Select(json, ".user.profile.name:s");  // String
cJSON *age = cJSON_Select(json, ".user.profile.age:n");    // Number
cJSON *tags = cJSON_Select(json, ".user.tags:a");          // Array
cJSON *meta = cJSON_Select(json, ".user.meta:o");          // Object

// Access array elements:
cJSON *first = cJSON_Select(json, ".items[0].name:s");
cJSON *third = cJSON_Select(json, ".items[2]:o");

// Use * to pass indices or field names as arguments:
int idx = 5;
cJSON *item = cJSON_Select(json, ".items[*].name:s", idx);

char *field = "email";
cJSON *email = cJSON_Select(json, ".user.*:s", field);

// Type specifiers:
// :s = string, :n = number, :a = array, :o = object, :b = boolean, :! = null

// After selecting, access values:
if (name) printf("Name: %s\n", name->valuestring);
if (age) printf("Age: %d\n", (int)age->valuedouble);

// Don't forget to free:
cJSON_Delete(json);
```

## SDS strings

SDS (Simple Dynamic Strings) is a string library that makes C string handling much safer and more convenient. SDS strings are binary-safe, know their length, and automatically manage memory. The full documentation is at https://github.com/antirez/sds but here are the most common operations:

```c
// Create strings
sds s = sdsempty();              // Empty string
sds s = sdsnew("Hello");         // From C string
sds s = sdsnewlen(buf, len);     // From buffer with length
sds s = sdsfromlonglong(12345);  // From integer

// Concatenate (these may reallocate, so always use the return value)
s = sdscat(s, " World");         // Append C string
s = sdscatlen(s, buf, len);      // Append buffer
s = sdscatprintf(s, " %d", 42);  // Append formatted

// Length and access
size_t len = sdslen(s);          // Get length (O(1), no strlen!)
s[0] = 'h';                      // Direct character access is fine

// Modify
s = sdstrim(s, " \t\n");         // Trim characters from both ends
sdsrange(s, 0, 4);               // Keep only characters 0-4

// Split
int count;
sds *tokens = sdssplitargs("hello world", &count);
// Use tokens[0], tokens[1], etc.
sdsfreesplitres(tokens, count);  // Free the array and all strings

// Free
sdsfree(s);
```

The key thing to remember: SDS strings are compatible with C strings for reading (you can pass them to `printf`, `strcmp`, etc.) but you must use SDS functions to modify them, and always use the return value since the pointer may change on reallocation.
