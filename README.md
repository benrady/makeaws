### Why in the world would you do this?

Make is good at two things:

*   Resolving dependent tasks, possibly in parallel
*   Transforming stuff into other stuff

If we think of the AWS command line tool as a sort of compiler, and the configuration files it uses as source code, then the output from those commands could be thought of as a sort of binary artifact...one that represent the assets that are created in AWS when the commands are run.

I know this is insane but stay with me for a minute.

Let's say I want to create a S3 bucket. I've defined the configuration for the bucket in a file called bucket.json. The AWS CLI command to do this is:

<pre>    $ aws s3api create-bucket --cli-input-json file://bucket.json
  </pre>

The output from this command happens to be a small JSON string that includes the bucket location. If we take the output of this command and redirect it to a file,

<pre>    $ aws s3api create-bucket --cli-input-json file://bucket.json > bucket
  </pre>

then we're simply taking an input file and transformed it into an output. Which is not unlike something you'd do with gcc:

<pre>    $ gcc -o bucket bucket.c
  </pre>

Many a makefile has been built upon this principle.

<pre>    CC=gcc
    CFLAGS=-I.
    DEPS = bucket.h
    OBJ = bucket.o

    %.o: %.c $(DEPS)
            $(CC) -c -o $@ $< $(CFLAGS)

    bucket: $(OBJ)
            gcc -o $@ $^ $(CFLAGS)
  </pre>

Of course, simply running commands for us isn't a great use of make. You could just as easily do this with a shell script. But what if we wanted to take this a step further, and turn this S3 bucket into a static website? Well, AWS lets you do that easily with the `put-bucket-website` command, but the bucket has to exist already. This is where make starts to shine.

Let's say we have a make target that creates our bucket.

<pre>    bucket: bucket.json
            @aws s3api create-bucket --cli-input-json bucket.json > bucket
  </pre>

We can add another target to make the website that depends on that target.

<pre>    website: bucket website.json
            @aws s3api put-bucket-website --cli-input-json website.json > website_info
  </pre>

Now, we can create the bucket and turn it into a static website by running a single command.

<pre>    $ make website
  </pre>

If we had previously run the `make bucket` command, make would figure out that's already done, and just make the website. If we hadn't run either, it would do both, and if we had already done both, it would do nothing.

But wait, there's more!

What if we wanted to change a setting for our website? Perhaps adding https support, or changing the default error page? Well all we have to do is change the configuration file, website.json, and re-run `make website`. That will update the website settings, but not try to re-create the bucket. And if we're unsure about what effects these changes will have, we can always use make's `--dry-run` flag to see what it will do before doing it.

### Wait, this actually seems cool. Why is this a bad idea again?

After you go creating all these services in AWS, you need to keep track of which ones were created, and why, and when....so you don't accidentally go making them all over again. If you only ever do this on one computer, then make is a perfect solution.

However, if you were to move these files to another computer using a method that doesn't keep track of the file modification times (say, via source control), then all the dependency management that make provides will get very confused about the order that things have been done. It may try to re-run tasks that are already up to date...or not...it sort of depends on the exact file modification time of the copied files. The result of running these AWS CLI commands accidentally can be disastrous, so that's not a risk we should be willing to take.

There is a sort-of workaround for this in make called "order only dependencies." When you have dependencies that are modified even when they are not updated, you can add the dependencies to a target following a pipe "|", and make will only run those dependent targets if they are missing, rather than if they're too old. Great! Problem solved, right?

Well, by using order only dependencies, we've gone and completely subverted the reason we were using make in the first place. If I now go change the configuration for my website, instead of make figuring out what needs to be updated, _I_ have to manually remove the output from that target to force make to re-run the task. And then I need to do the same for all the downstream tasks too, and to do that I have to figure out what they all are.

So, until the most popular source control systems start preserving timestamps, this is a terrible, horrible, no-good bad idea.
