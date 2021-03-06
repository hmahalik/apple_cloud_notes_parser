# Apple Cloud Notes Parser
By: Jon Baumann, [Ciofeca Forensics](https://www.ciofecaforensics.com)

## About

This program is a parser for the current version of Apple Notes data syncable with iCloud as seen on Apple handsets in iOS 9 and later. 
This program was made as an update to the [previous Perl script](https://github.com/threeplanetssoftware/apple_cloud_notes_parser) which did not well handle the protobufs used by Apple in Apple Notes. 
That script and this program are needed because data that was stored in plaintext in the versions of Apple's Notes prior to iOS 9 in its `notes.sqlite` database is now gzipped before storage in the iCloud Notes database `NoteStore.sqlite` and the amount of embedded objects inside of Notes is far higher.
While the data is not necessarily encrypted, although some is using the password feature, it is not as searchable to the examiner, given its compressed nature. 
This program intends to make the plaintext stored in the note and its embedded attachments far more usable.

This program was implemented in Ruby. 
The classes underlying this represent all the necessary features to write other programs to interface with an Apple Notes backup, including exporting data to another format, or writing better search functions. 
In addition, this program and its classes attempts to abstract away the work needed to understand the type of backup and how it stores files. 
While examiners must understand those backups, this will provide its own internal interfaces for identifying wheremedia files are kept. 
For example, if the backup is from iTunes, this program will use the Manifest.db to identify the hashed name of the included file, and copy out/rename the image to the appropriate name, without the examiner having to do that manually.

## Features

This program will:
1. Parse legacy (pre-iOS9) Notes files (but those are already plaintext, so not much to be gained)
2. Parse iOS 9-13 Cloud Notes files
3. ... generating CSV roll-ups of each account, folder, note, and embedded object within them
4. ... rebuilding the notes as an HTML file to browse and see as they would be displayed on the phone
5. ... amending the NoteStore.sqlite database to include plaintext and decompressed objects to interact with in other tools
6. ... from iTunes logical backups, physical backups, or single files
7. ... displaying tables as actual tables and ripping the embedded images from the backup and putting them into a folder with the other output files for review.

## Usage

### Base

This program is run by Ruby on a command line, using either `rake` or `ruby`. 
The easiest use, which is the same as the original Perl script, is to drop your exported `NoteStore.sqlite` into the same directory as this program and then run `rake` (which is the same as running `ruby notes_cloud_ripper.rb --file NoteStore.sqlite`). 

If you are more comfortable with the command line, you can point the program anywhere you would like on your computer and specify the type of backup you are looking at, such as: `ruby notes_cloud_ripper.rb --itunes-dir ~/.iTunes/backup1/` or `ruby notes_cloud_ripper.rb --file ~/.iTunes/backup1/4f/4f98687d8ab0d6d1a371110e6b7300f6e465bef2`. 
The benefit of pointing at full backups is this program can pull files out of them as needed, such as drawings and pictures.

### Options

The options that are currently supported are:
1. `-i | --itunes-dir DIRECTORY`: Tells the program to look at an iTunes backup folder.
2. `-f | --file FILE`: Tells the program to look only at a specific file that is identified. 
3. `-p | --physical DIRECTORY`: Tells the program to look at a physical backup folder.
4. `-o | --output-dir DIRECTORY`: Changes the output folder from the default `./output` to the specified one.
5. `-h | --help`: Prints the usage information.

## How It Works

### iTunes backup (-i option)

For backups created with iTunes MobileSync, that include a wide range of hashed files inside of folders named after the filename, this program expects to be given the root folder of that backup. With that, it will compute the path to the NoteStore.sqlite file. If it exists, that file will be copied to the output directory and the copy, not the original, will be opened.

For example, if you had an iTunes backup located in `/home/user/phone_rips/iphone/[deviceid]/` (Which means the Manifest.db is located at `/home/whatever/phone_rips/iphone/[deviceid]/Manifest.db`) you would run: `ruby notes_cloud_ripper.rb -i /home/user/phone_rips/iphone/[deviceid]/`

### Physical backup (-p option)

For backups created with a full file system (or at least the `/private` directory) from your tool of choice. This program expects to be given the root folder of that backup. With that, it will compute the path to the NoteStore.sqlite file. If it exists, that file will be copied to the output directory and the copy, not the original, will be opened.

For example, if you had a physical backup located in `/home/user/phone_rips/iphone/physical/` (Which means the phone's `/private` directory is located at `/home/whatever/phone_rips/iphone/physical/private/`) you would run: `ruby notes_cloud_ripper.rb -p /home/user/phone_rips/iphone/physical`

### Single File (-f option)

For single file "backups", this program expects to be given the path of the NoteStore.sqlite file directly, although filename does not matter. If it exists, that file will be copied to the output directory and the copy, not the original, will be opened.

For example, if you had a NoteStore.sqlite file located in `/home/user/phone_rips/iphone/files/NoteStore.sqlite` you would run: `ruby notes_cloud_ripper.rb -f /home/user/phone_rips/iphone/files/NoteStore.sqlite`

### All Versions

Once the NoteStore file is opened, the program will create new AppleNotesAccount, AppleNotesFolder, and AppleNote objects based on the contents of that file. 
For each note, it takes the gzipped blob in the ZDATA field, gunzips it, and parses the protobuf that is inside. 
It will then add the plaintext from the protobuf of each note back into the NoteStore.sqlite file's `ZICNOTEDATA` table as a new column, `ZPLAINTEXTDATA` and create AppleNotesEmbeddedObject objects for each of the embedded objects it identifies.

### Output 

All of the output from this program will go into the output folder (it will be created if it doesn't exist), which defaults to `[location of this program]/output`. 
Within that folder, sub-folders will be created based on the current date and time, to the second. 
For example, if this program was run on December 17, 2019 at 10:24:28 local, the output for that run would be in `[location of this program]/output/2019_12_17-10_24_28/`. 
If the type of backup used has the original files referenced in attached media, this program will copy the file into the output directory, under `[location of this program]/output/[date of run]/files`, and beneath that following the file path on disk, relative to the Notes application folder. 
This program will produce four CSV files summarizing the information stored in `[location of this program]/output/[date of run]/csv`: `note_store_accounts.csv`, `note_store_embedded_objects.csv`, `note_store_folders.csv`, and `note_store_notes.sqlite`. 
Finally, it will produce an HTML dump of the notes to reflect the text and table formatting which may be meaningful in `[location of this program]/output/[date of run]/html`.

Because Apple devices often have more than one version of Noes, it is important to note (pun intended) that all of the output is suffixed by a number, starting at 1, to identify which of the backups it corresponds to. 
In all cases where more than one is found, care is taken to produce output that assigns the suffix of 1 for the modern version, and the suffix of 2 for the legacy version. 

## Requirements

### Ruby Version

This program has been tested with the following versions of Ruby:

|Ruby Version| OS | Status |
|------------|----|--------|
|2.3.0|Debian-based Linux|:heavy_check_mark:|
|2.3.1|Debian-based Linux|:heavy_check_mark:|
|2.4.3|Debian-based Linux|:heavy_check_mark:|
|2.5.1|Debian-based Linux|:heavy_check_mark:|
|2.6.5|Debian-based Linux|:heavy_check_mark:|
|2.4.3|macOS 10.13|:heavy_check_mark:|
|2.5.1|macOS 10.13|:heavy_check_mark:|
|2.6.5|Windows 8|:heavy_check_mark:|
|2.7.0|Any|:x:|

### Gems

This program requires the following Ruby gems which can be installed by running `bundle install` or `gem install [gemname]`:
1. fileutils
2. google-protobuf
   1. Note: This does not yet work with Ruby 2.7, use any of the 2.6 varients, or lower
3. sqlite3
4. zlib

## Installation

Below are instructions, generally preferring the command line, for each of Linux, Mac, and Windows. The user can choose to use Git if they want to be able to keep up with changes, or just download the tool once, you do not need to do both. On each OS, you will want to:

1. Install Ruby (at least version 2.3.0), its development headers, and bundler if not already installed.
2. Install development headers for SQLite3 if not already installed.
3. Get this code
   1. Clone this repository with Git or
   2. Download the Zip file and unzip it
4. Enter the repository's directory.
5. Use bundler to install the required gems.
6. Run the program (see Usage section)!

### Debian-based Linux (Debian, Ubuntu, Mint, etc)

#### With Git (If you want to stay up to date)

```bash
sudo apt-get install build-essential libsqlite3-dev zlib1g-dev git ruby-full ruby-bundler
git clone https://github.com/threeplanetssoftware/apple_cloud_notes_parser.git
cd apple_cloud_notes_parser
bundle install
```

#### Without Git (If you want to download it every now and then)

```bash
sudo apt-get install build-essential libsqlite3-dev zlib1g-dev git ruby-full ruby-bundler
curl https://codeload.github.com/threeplanetssoftware/apple_cloud_notes_parser/zip/master -o apple_cloud_notes_parser.zip
unzip apple_cloud_notes_parser.zip
cd apple_cloud_notes_parser-master
bundle install
```
### Red Hat-based Linux (Red Hat, CentOS, etc)

#### With Git (If you want to stay up to date)

```bash
sudo yum groupinstall "Development Tools" 
sudo yum install sqlite sqlite-devel zlib zlib-devel openssl openssl-devel ruby ruby-devel rubygem-bundler
git clone https://github.com/threeplanetssoftware/apple_cloud_notes_parser.git
cd apple_cloud_notes_parser
bundle install
sudo gem pristine sqlite3 zlib
```

#### Without Git (If you want to download it every now and then)

```bash
sudo yum groupinstall "Development Tools" 
sudo yum install sqlite sqlite-devel zlib zlib-devel openssl openssl-devel ruby ruby-devel rubygem-bundler
curl https://codeload.github.com/threeplanetssoftware/apple_cloud_notes_parser/zip/master -o apple_cloud_notes_parser.zip
unzip apple_cloud_notes_parser.zip
cd apple_cloud_notes_parser-master
bundle install
sudo gem pristine sqlite3 zlib
```

### macOS 

#### With Git (If you want to stay up to date)

```bash
git clone https://github.com/threeplanetssoftware/apple_cloud_notes_parser.git
cd apple_cloud_notes_parser
bundle install
```

#### Without Git (If you want to download it every now and then)

```bash
curl https://codeload.github.com/threeplanetssoftware/apple_cloud_notes_parser/zip/master -o apple_cloud_notes_parser.zip
unzip apple_cloud_notes_parser.zip
cd apple_cloud_notes_parser-master
bundle install
```

### Windows

1. Download the 2.6.5 64-bit [RubyInstaller with DevKit](https://github.com/oneclick/rubyinstaller2/releases/download/RubyInstaller-2.6.5-1/rubyinstaller-devkit-2.6.5-1-x64.exe) (Google Protobuf isn't yet working on 2.7)
2. Run RubyInstaller using default settings.
3. Download the latest [SQLite amalgamation souce code](https://sqlite.org/2019/sqlite-amalgamation-3300100.zip) and [64-bit SQLite Precompiled DLL](https://sqlite.org/2019/sqlite-dll-win64-x64-3300100.zip)
4. Install SQLite
   1. Create a folder C:\sqlite
   2. Unzip the source code into C:\sqlite (you should now have C:\sqlite\sqlite3.c and C:\sqlite\sqlite.h, among others)
   3. Unzip the DLL into C:\sqlite (you should now have C:\sqlite\sqlite3.dll, among others)
5. Download [this Apple Cloud Notes Parser as a zip archive](https://codeload.github.com/threeplanetssoftware/apple_cloud_notes_parser_ruby/zip/master)
6. Unzip the Zip archive
7. Launch a powershell window and navigate to where you unzipped the archive
8. Execute the following commands (these set the PATH so SQLite files can be found install SQLite's Gem specifically pointing to them, and then installs the rest of the gems):

```powershell
$env:Path += ";C:\sqlite"
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\sqlite", "User")
gem install sqlite3 --platform=ruby -- --with-sqlite-3-dir=C:/sqlite --with-sqlite-3-include=C:/sqlite
bundle install
```

## Folder Structure

For reference, the structure of this program is as follows:

```
apple_cloud_notes_parser
  |
  |-lib
  |  |
  |  |-notestore_pb.rb: Protobuf representation generated with protoc
  |  |-Apple\*.rb: Ruby classes dealing with various aspects of Notes
  |
  |-output (created after run)
  |  |
  |  |-[folders for each date/time run]
  |     |
  |     |-csv: This folder holds the CSV output
  |     |-files: This folder holds files copied out of the backup, such as pictures
  |     |-html: This folder holds the generated HTML copy of the Notestore
  |     |-Manifest.db: If run on an iTunes backup, this is a copy of the Manifest.db
  |     |-NoteStore.sqlite: If run on a modern version, this copy of the target file will include plaintext versions of the Notes
  |     |-notes.sqlite: If run on a legacy version, this copy is just a copy for ease of use
  |
  |-.gitignore
  |-.travis.yml
  |-Gemfile
  |-LICENSE
  |-README.md
  |-Rakefile
  |-notes_cloud_ripper.rb: The main program itself
```

## FAQ

#### Why do I get a "No such column" error?

Example: `/var/lib/gems/2.3.0/gems/sqlite3-1.4.1/lib/sqlite3/database.rb:147:in 'initialize': no such column: ZICCLOUDSYNCINGOBJECT.ZSERVERRECORDDATA (SQLite3::SQLException)`

Apple changed the format of its Notes database in different versions of iOS. While the supported versions *should* be supported, interesting cases may come up. Please open an issue and include the stack trace and the following information:
* iOS version (including any versions it may have upgraded from)
* The results of `SELECT name,sql FROM sqlite_master WHERE type="table"` when the database is open in sqlitebrowser (or your editor of choice). This can be in any columned format (Excel, CSV, SQL, etc)
* If possible, the database file directly (I can receive it through other means if it needs to stay confidential). If this is possible, the above results are not needed.

#### Why do I get a Load Error from google-protobuf immediately upon execution?

Example: `cannot load such file -- google/2.7/protobuf_c (LoadError)`

The current version of the google-protobuf gem (3.11.2) was released days before Ruby 2.7. Once the next version of the gem is released, it will hopefully go away. In the meantime, feel free to use Ruby 2.6.5 or previous verious while you wait.

#### Why Ruby instead of Python or Perl?

Programming languages are like human languages, there are many and which you choose (for those with multiple) can largely be a personal preference, assuming mutual intelligibility. I chose Ruby as the previous Perl code was a nice little script, but I wanted a bit more substance behind with with solid object oriented programming principals. For those new to Ruby, hopefully these classes will give you an idea of what Ruby has to offer and spark an interest in trying a new language.

## Known Bugs

1. Google Protobuf throws errors on Windows with Ruby 2.7, likely the gem isn't updated yet.
2. Encrypted notes are not yet decrypted (not technically a bug, but worth noting).

## Acknowledgements

* Jmendeth's [protobuf-inspector](https://github.com/jmendeth/protobuf-inspector) drove most of my analysis into the Notes protobufs.
* Previous work by [dunhamsteve](https://github.com/dunhamsteve/notesutils/blob/master/notes.md) proved invaluable to finally understanding the embedded table aspects.
