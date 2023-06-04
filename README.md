
# yt-fts - Youtube Full Text Search 
`yt-fts` is a simple python script that uses yt-dlp to scrape all of a youtube channels subtitles
and load them into an sqlite database that is searchable from the command line. It allows you to
query a channel for specific key word or phrase and will generate time stamped youtube urls to
the video containing the keyword. 

- [Blog Post](https://notjoemartinez.com/blog/youtube_full_text_search/)
- [Semantic Search](#Semantic-Search-via-OpenAI-embeddings-API) (Experimental)

## Installation 

**pip**
```bash
pip install yt-fts
```

**from source**
```bash
git clone https://github.com/NotJoeMartinez/yt-fts
python3 -m venv .env
source .env/bin/activate
pip install -r requirements.txt
python3 -m yt-fts
```

## Dependencies 
This project requires [yt-dlp](https://github.com/yt-dlp/yt-dlp) installed globally. Platform specific installation instructions are available on the [yt-dlp wiki](https://github.com/yt-dlp/yt-dlp/wiki/Installation). 

**pip**
```bash
python3 -m pip install -U yt-dlp
```
**MacOS/Homebrew**
```bash
brew install yt-dlp
```
**Windows/winget**
```bash
winget install yt-dlp
```

## Usage 
```
Usage: yt-fts [OPTIONS] COMMAND [ARGS]...

Options:
  --version  Show the version and exit.
  --help     Show this message and exit.

Commands:
  delete              Delete a channel and all its data.
  download            Download subtitles from a specified YouTube channel.
  export              Export search results from a specified YouTube...
  generate-embeddings  Generate embeddings for a channel using OpenAI's...
  list                Lists channels saved in the database.
  search              Search for a specified text within a channel, a...
  semantic-search     Semantic search for specified text.
  update              Updates a specified YouTube channel.
```

## `download`
Download subtitles 
```
Usage: yt-fts download [OPTIONS] CHANNEL_URL

  Download subtitles from a specified YouTube channel.

  You must provide the URL of the channel as an argument. The script will
  automatically extract the channel id from the URL.

Options:
  --channel-id TEXT         Optional channel id to override the one from the
                            url
  --language TEXT           Language of the subtitles to download
  --number-of-jobs INTEGER  Optional number of jobs to parallelize the run
```

### Examples:

**Basic download by url**

```bash
yt-fts download "https://www.youtube.com/@TimDillonShow/videos"
```

**Multithreaded download**

```bash
yt-fts download --number-of-jobs 6 "https://www.youtube.com/@TimDillonShow/videos"
```

**specify channel id**

If `download` fails you can manually input the channel id with the `--channel-id` flag.
The channel url should still be an argument 

```bash
yt-fts download --channel-id "UC4woSp8ITBoYDmjkukhEhxg" "https://www.youtube.com/@TimDillonShow/videos" 
```

**specify language**

Languages are represented using [ISO 639-1](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) language codes 

```bash
yt-fts download --language de "https://www.youtube.com/@TimDillonShow/videos" 
```

## `list`
List downloaded channels 
```
Usage: yt-fts list [OPTIONS]

  Lists channels saved in the database.

  The (ss) next to channel name indicates that semantic search is enabled for
  the channel.

Options:
  --channel TEXT  Optional name or id of the channel to list
```

```bash
yt-fts list
```

output:
```
  id    count  channel_name         channel_url
----  -------  -------------------  ----------------------------------------------------
   1      265  The Tim Dillon Show  https://youtube.com/channel/UC4woSp8ITBoYDmjkukhEhxg
   2      688  Lex Fridman (ss)     https://youtube.com/channel/UCSHZKyawb77ixDdsGog4iWA
   3      434  Traversy Media       https://youtube.com/channel/UC29ju8bIPH5as8OGnQzwJyA
```

## `search`
Search saved subtitles 
```
Usage: yt-fts search [OPTIONS] SEARCH_TEXT

  Search for a specified text within a channel, a specific video, or across
  all channels.

Options:
  --channel TEXT  The name or id of the channel to search in. This is required
                  unless the --all or --video options are used.
  --video TEXT    The id of the video to search in. This is used instead of
                  the channel option.
  --all           Search in all channels.
```

- The search string does not have to be a word for word and match 
- Use Id if you have channels with the same name or channels that have special characters in their name 
- Search strings are limited to 40 characters. 

### Examples:

**Search by channel**

```bash
yt-fts search "life in the big city" --channel "The Tim Dillon Show"
# or 
yt-fts search "life in the big city" --channel 1  # assuming 1 is id of channel
```
output:
```
The Tim Dillon Show: "164 - Life In The Big City - YouTube"

    Quote: "van in the driveway life in the big city"
    Time Stamp: 00:30:44.580
    Video ID: dqGyCTbzYmc
    Link: https://youtu.be/dqGyCTbzYmc?t=1841
```

**Search all channels**

```bash
yt-fts search "text to search" --all
```

**Search in video**

```bash
yt-fts search "text to search" --video [VIDEO_ID]
```

**Advanced Search Syntax**

The search string supports sqlite [Enhanced Query Syntax](https://www.sqlite.org/fts3.html#full_text_index_queries).
which includes things like [prefix queries](https://www.sqlite.org/fts3.html#termprefix) which you can use to match parts of a word.  

```bash
yt-fts search "rea* kni* Mali*" --channel "The Tim Dillon Show" 
```
output:
```
The Tim Dillon Show: "#200 - Knife Fights In Malibu | The Tim Dillon Show - YouTube"

    Quote: "real knife fight down here in Malibu I"
    Time Stamp: 00:45:39.420
    Video ID: e79H5nxS65Q
    Link: https://youtu.be/e79H5nxS65Q?t=2736
```

## `export`
Export search results to csv. Exported csv will have `Channel Name,Video Title,Quote,Time Stamp,Link` as it's headers
```
Usage: yt-fts export [OPTIONS] SEARCH_TEXT [CHANNEL]

  Export search results from a specified YouTube channel or from all channels
  to a CSV file.

  The results of the search will be exported to a CSV file. The file will be
  named with the format "{channel_id or 'all'}_{TIME_STAMP}.csv"

Options:
  --all   Export from all channels
```

**Examples:**
```bash
yt-fts export "life in the big city" "The Tim Dillon Show"
```

You can export from all channels in your database as well
```bash
yt-fts export "life in the big city" --all
```

## update
Will update a channel with new subtitles if any are found. 
```
Usage: yt-fts update [OPTIONS]

  Updates a specified YouTube channel.

  You must provide the ID of the channel as an argument. Keep in mind some
  might not have subtitles enabled. This command will still attempt to
  download subtitles as subtitles are sometimes added later.

Options:
  --channel TEXT            The name or id of the channel to update.
                            [required]
  --language TEXT           Language of the subtitles to download
  --number-of-jobs INTEGER  Optional number of jobs to parallelize the run
```

## `delete` 
Will delete a channel from your database 
```
Usage: yt-fts delete [OPTIONS]

  Delete a channel and all its data.

  You must provide the name or the id of the channel you want to delete as an
  argument.

  The command will ask for confirmation before performing the deletion.

Options:
  --channel TEXT  The name or id of the channel to delete in  [required]
```

**Examples:**

```bash
yt-fts delete "The Tim Dillon Show"
# or
yt-fts delete 1 
```

--- 
# Semantic Search via OpenAI embeddings API 
The following commands are a work in progress but should enable semantic search. 
This requires that you have an openAI API key which you can learn more about that [here](https://platform.openai.com/docs/api-reference/introduction). 

**Limitations**

Keep in mind that generating embeddings will substantially grow the size of your subtitles database and will run slower due to the limitations of working with vectors in sqlite. When running semantic
searches for the first time, API access is still required to generate embeddings for the search string.
These search string embeddings are saved to a history table and won't require additional api requests
after. 

### `semantic-search` 
```
Usage: yt-fts semantic [OPTIONS] SEARCH_TEXT

  Semantic search for specified text.

  Before running this command, you must generate embeddings for the channel
  using the generate-embeddings command. This command uses OpenAI's embeddings 
  API to search for specified text. An OpenAI API key must be set as an
  environment variable OPENAI_API_KEY.

Options:
  --channel TEXT   channel name or id to search in
  --all            Search all semantic search enabled channels
  --limit INTEGER  top n results to return
```

### `generate-embedings`
```
Usage: yt-fts generate-embedings [OPTIONS]

  Generate embeddings for a channel using OpenAI's embeddings API.

  Requires an OpenAI API key to be set as an environment variable
  OPENAI_API_KEY.

Options:
  --channel TEXT       The name or id of the channel to generate embeddings
                       for
  --open-api-key TEXT  OpenAI API key. If not provided, the script will
                       attempt to read it from the OPENAI_API_KEY environment
                       variable.
```