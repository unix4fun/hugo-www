# hugo-www


## Useful Links:
- [hugo](https://gohugo.io)


## Markdown examples:

### basic formatting :

- ~~This was mistaken text~~
- **This text is _extremely_ important**

### command line:

    $ go get github.com/spf13/nitro


### footnote
This is a footnote.[^1]
[^1]: the footnote text.


### mention
Cat
: Fluffy animal everyone likes

Internet
: Vector of transmission for pictures of cats

### code highlight
``` go
func getTrue() bool {
    return true
}
```

### array
```
Name    | Age
--------|------
Bob     | 27
Alice   | 23
```

## Hugo post:
	$ cd hugo-www-repo
	$ 
	$ hugo new post/this-is-my-new-post.md
	$ vim content/post/this-is-my-new-post-md

Do your magic, check the post rendering _hugo server_ locally, if it's all ok.
	$ hugo

Then push the updated to the github.unix4fun.io repo, you're done.


## TODO:
- shortcodes.
- more examples of markdown, because i suck.
