# iMessage EDA and Analysis


Text conversations are a treasure trove of data about who we communicate with, what we talk about, and more. Let's dig into what these conversations can tell us!

If you're an iPhone / Mac user, you're sitting on a treasure trove of interesting personal connection and communication data just waiting to be given a voice. I personally used this data to put together a book for my SO as a gift for Valentine's day, but the possabilities are endless!

### Finding your Messages
iPhones and Macs store iMessage content in an sqlite database. This isn't the easiest thing to get off of an iPhone (though it is possible), but it is very easy to pull from a Mac so long as you have iMessage enabled, and iMessage in iCloud turned on (this is a setting on both iPhones and Macs). 

By default terminal applications don't have full access to the OSX filesystem, so you can either go into settings and enable Privacy & Security -> Full Disk Access -> Your Terminal client of choice, or you can just use Finder to snag the chat database by navigating to ~/Library/Messages/ and copying chat.db somplace you can access it.

{{< image src="gotofolder.jpg" caption="Pull up Go to Folder" linked=false >}} {{< image src="typegoto.png" caption="Navigate to Messages" linked=false >}}

