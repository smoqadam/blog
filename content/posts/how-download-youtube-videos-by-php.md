---
title: "How to write a PHP Youtube video downloader"
date: 2020-04-16T21:37:50+08:00
draft: false
---

PHP is not the best language for writing a download manager, but in this article, we are going to learn how we can download videos from Youtube. Because PHP does not support multithreading, we cannot make a proper download manager that split the file into several chunks and download it concurrently. There are some ways to add multithreading to PHP, but for the sake of this article, we won't go through those ways and will stick to single threading.

<!--more -->


At the end of this article, we have a command called `dl`, and it gets a Youtube video URL to download.

```bash
$ php index.php dl https://www.youtube.com/watch?v=XXXXXXXXX

Video formats
1: video/mp4; codecs="avc1.640028" - 39013359
2: video/webm; codecs="vp9" - 35592706
3: video/mp4; codecs="av01.0.08M.08" - 33395130
4: video/mp4; codecs="avc1.4d401f" - 9282564
5: video/webm; codecs="vp9" - 11742153
6: video/mp4; codecs="av01.0.05M.08" - 19186717
7: video/mp4; codecs="avc1.4d401e" - 5689359
8: video/webm; codecs="vp9" - 6911794
9: video/mp4; codecs="av01.0.04M.08" - 8974711
10: video/mp4; codecs="avc1.4d401e" - 4401516

Which format do you want to download? (default: 1):10

Connected...
Mime-type: video/mp4
Filesize: 4401516
[==============================================>] 100% ( 4/ 4 mb)
```


### What we will learn by the end of this article:

- How to create a command-line application
- How to use the Factory design pattern
- How to get information about a Youtube video such as formats, titles, etc.
- How to download a file with PHP with a progress bar



### Create the project

create a new directory and run `composer init` inside it:

```bash
$ mkdir youtueb-downloader && cd youtueb-downloader
$ composer init
```

After running `composer init‍` you need to answer a few questions. This command creates a `composer.json` file inside `youtube-downloader` directory.



### Install dependencies

```bash
$ composer require symfony/console
$ composer require smoqadam/youtube-video-info:dev-master
```

- `symfony/console` is a great package that helps us to create a command-line application and make it easier to handle input and outputs.

- `smoqadam/youtube-video-info` gives us information about a Youtube video. The information such as video formats, subtitles, download URL, title, etc.


### autoloading

Create a directory and call it `src` we will put all of our classes inside this directory. For taking advantage of composer autloading, open `composer.json` and add `autoload` sectoin such as below:

```json
{
    "autoload": {
        "psr-4": {
            "Downloader\\": "src"
        }
    }
}
```

`Downloader` is our chosen namespace for this project, and whenever the composer wants to load a class within this namespace, it looks inside the `src` directory.

You can find more information about PSR4 [here](LINK).


### Handle Command Line arguments

 
Create a `Download.php` file inside the `src/Commands` directory and with the following content:

```php
<?php
// src/Commands/Download.php
namespace Downloader\Commands;

use Downloader\Downloader;
use Downloader\Factory;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class Download extends Command
{
    protected static $defaultName = "dl";

    private $arg = 'url';

    protected function configure()
    {
        $this->setDescription("URL")->addArgument($this->arg, InputArgument::REQUIRED);
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $url = $input->getArgument($this->arg);
        echo $outpu->write("<info>{$url}</info>")
        return 0;
    }
}
```

`Download` class extended `Command` from the `symfony/console` package, which gives us many features to handle command-line input and outputs.

- `$defaultName` is the name of our command.
- `configure` method is using to config our command, such as add description, add arguments, options, and so on.
- Inside the `configure` method, we can configure the `dl` command, for example, add a description, add arguments, options, and many more.
- And finally, `execute` will call when we run the `dl` command from CLI. It receives two parameters, `$input` for handling the inputs such as options and arguments, and `$output` for managing the output. As you can see, we used `<info></info>` tags in the write method to colorize the output.

Let's make our `dl` command work by creating a `index.php` file under root directory:

```php
<?php
// index.php

use Downloader\Commands\Download;
use Symfony\Component\Console\Application;

require_once 'vendor/autoload.php';

// Symfony/console application
$app = new Application();
$app->add(new Download());
$app->run();

```

`Application` is the main class responsible for handling the command line application, so to make our `dl` command work, we must use the `add` method to pass a new instance of `Download` class to it.

Now, open your terminal and run the following command:

```bash
$ PHP index.php dl http://example.com
```
you should see http://example.com in green color as output.


## Downloader

In this section, we are going to make a downloader class. This class receives a URL and a file name, and it will download the file for us. But before we create our Downloader class, let's think about how we can make it extendable? By extendable, I mean how we can have a Downloader that downloads files from different sources such as Youtube or Vimeo videos or may a direct link.

We know that Youtube videos have different qualities, and each quality has a different download link. So before we can download a video, we must get all qualities and ask users which quality they want to download. But this process will be different if they're going to download from Vimeo or a direct link.

In the SOLID principle, **S** is for **Single responsibility**, which means each class must do only one thing. With that explanation, we need to have different classes for different sources. 

To achieve this, we can use **Factory** design pattern to instantiate a new object based on the URL that the user wants to download.

For example, if it was a Youtube URL, it will return a new object from `Youtube` class and so on. 

`Factory.php` class under `src/` directory would be something like this:

```php
<?php

// src/Factory.php

namespace Downloader;

class Factory
{
    public static function make(string $url): ProviderAbstract
    {
        switch (true) {
            case preg_match('%(?:youtube(?:-nocookie)?\.com/(?:[^/]+/.+/|(?:v|e(?:mbed)?)/|.*[?&]v=)|youtu\.be/)([^"&?/ ]{11})%i', $url, $matches):
                return new Youtube($url);
            default:
                throw new Exception("Download link does not support");
        }
    }
}
```
**Factory** Design pattern is creational pattern that create a different object based on some conditions. In our case, we checked the `$url` and if it was a Yoututbe link, it returns a new instance of `Yoututbe` class, otherwise it throws an `Exception`.


### Handle different sources

As we saw earlier, we decided to let our app download from different sources, but these sources or providers must follow some rules to prevent chaos and make it easier to extend and debug. We define our rules inside `ProviderAbstract`, and every provider (e.g., Youtube class) must extend from this class, and they must implement the `init` method.

#### ProviderAbstract

```php
// src/ProviderAbstract.php
<?php

namespace Downloader;
abstract class ProviderAbstract
{
    protected $url;
    protected $name;

    public function __construct($url)
    {
        $this->init($url);
    }

    public function getUrl(): string
    {
        return $this->url;
    }

    public function getName(): string
    {
        return $this->name;
    }

    abstract public function init(string $url): void;
}
```
Every provider must initialize and prepare the download link and output file name inside the `init` method.

### Youtube provider

Create a directory under `src` and call it `Providers` then create a file `Youtube.php` inside it:

```php
<?php
// src/Providers/Youtube.php

namespace Downloader\Providers;

use Downloader\ProviderAbstract;
use Smoqadam\Video;

class Youtube extends ProviderAbstract
{
    protected $pattern = '%(?:youtube(?:-nocookie)?\.com/(?:[^/]+/.+/|(?:v|e(?:mbed)?)/|.*[?&]v=)|youtu\.be/)([^"&?/ ]{11})%i';

    public function init(string $url): void
    {
        $video = new Video($this->getVideoId($url));
        // fetch all formats and qualities 
        $formats = $video->getFormats();
        print("\nVideo formats");
        // print a simple menu by each quality in a new line
        foreach ($formats as $index => $format) {
            printf("\n%d: %s - %d", $index + 1, $format->getMimeType(), $format->getSize());
        }

        // wait for user to enter index number associated to a menu item
        printf("\nWhich format do you want to download? (default: 1):");
        $i = readline();
        if (!$i) {
            $i = 1;
        }
        $format = $formats[$i - 1];
        // set url and name
        $this->url = $format->getUrl();
        $this->name = $video->getDetails()->getTitle();
    }

    private function getVideoId($url)
    {
        preg_match($this->pattern, $url, $match);
        return $match[1];
    }
}


```


`Youtube` class extended from `ProviderAbstract`, which means it must implement the `init` method. Inside `init`, we took advantage of `smoqadam/youtube-video-info` package to parse a youtube video. Then created a simple menu with each format in a new line. `readline` function receives an input which in our case it's an index of a menu item and whenever the user enters the number associated to the format, we filled up `$this->url` and `$this->name`.

 The menu would be something like this:

```bash
 1: video/mp4; codecs="avc1.640028" - 105345097
 2: video/webm; codecs="vp9" - 64217678
 3: video/mp4; codecs="avc1.64002a" - 113938770
 4: video/webm; codecs="vp9" - 107315439
 5: video/mp4; codecs="av01.0.09M.08" - 58147823

Which format do you want to download? (default: 1):
```

| Note: File sizes are in byte, you can convert to MB by yourself.

### Downloader

Here we are going to see how we can download a file and how to show a progress bar with PHP. Create a file under the `src` directory and call it `Downloader.php`:

```php
// src/Downloader.php
<?php

namespace Downloader;

class Downloader
{
    private $provider;

    public function __construct(ProviderAbstract $provider)
    {
        $this->provider = $provider;
    }

    /**
     * 
     */
    public function download()
    {
        $context = stream_context_create();
        stream_context_set_params($context, ['notification' => [\Downloader\Downloader::class, 'streamCallback']]);
        $handler = @fopen($this->provider->getUrl(), 'r', false, $context);
        if (!$handler) {
            throw new \InvalidArgumentException("File not found");
        }
        file_put_contents($this->provider->getName(), $handler);
    }

    /**
     * Stolen from https://www.php.net/manual/en/function.stream-notification-callback.php
     */
    private function streamCallback($notification_code, $severity, $message, $message_code, $bytes_transferred, $bytes_max)
    {
        static $filesize = null;
        switch ($notification_code) {
            case STREAM_NOTIFY_RESOLVE:
            case STREAM_NOTIFY_AUTH_REQUIRED:
            case STREAM_NOTIFY_COMPLETED:
            case STREAM_NOTIFY_FAILURE:
            case STREAM_NOTIFY_AUTH_RESULT:
                break;
            case STREAM_NOTIFY_REDIRECTED:
                echo "Being redirected to: ", $message, "\n";
                break;
            case STREAM_NOTIFY_CONNECT:
                echo "Connected...\n";
                break;
            case STREAM_NOTIFY_FILE_SIZE_IS:
                $filesize = $bytes_max;
                echo "Filesize: ", $filesize, "\n";
                break;
            case STREAM_NOTIFY_MIME_TYPE_IS:
                echo "Mime-type: ", $message, "\n";
                break;
            case STREAM_NOTIFY_PROGRESS:
                if ($bytes_transferred > 0) {
                    if (!isset($filesize)) {
                        printf("\rUnknown filesize.. %2d mb done..", $bytes_transferred / 1024 / 1024);
                    } else {
                        $length = (int)(($bytes_transferred / $filesize) * 100);
                        printf("\r[%-100s] %d%% (%2d/%2d mb)", str_repeat("=", $length) . ">", $length, ($bytes_transferred / 1024 / 1024), $filesize / 1024 / 1024);
                    }
                }
                break;
        }
    }
}
```


This class is not as scary as it looks. Just notice how we passed a `ProviderAbstract` class to constructor. We will talk about the `download` method in the next section.


#### What is Stream Context?

In PHP, [Stream Context](https://www.php.net/manual/en/stream.contexts.php) is using to enhance or modify a stream, and it can pass to most file handling functions. From the PHP website:

>  A context is a set of parameters and wrapper specific options which modify or enhance the behavior of a stream. Contexts are created using stream_context_create() and can be passed to most filesystem related stream creation functions (i.e. fopen(), file(), file_get_contents(), etc...). 


`stream_context_set_params` receives a stream context and an array of parameters. One of the parameters we can set is `notification` that is a function. This function will be called whenever our context state such changes. (see more: https://www.php.net/manual/en/function.stream-notification-callback.php) 

We passed the URL we want to download and the context we made earlier to `fopen`, it will open the remote file in a read mode, and whenever the state of the file changes, it will call `streamCallback` method. Inside the `streamCallback` method, we checked for the notification type and printed the proper information about each notification. 


Now open `sr/Command/Download.php` and change the `execute` method as below:

```php

<?php

namespace Downloader\Commands;

use Downloader\Downloader;
use Downloader\Factory;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class Download extends Command
{
    // ...

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $url = $input->getArgument($this->arg);
        $downloader = new Downloader(Factory::make($url));
        $downloader->download();
        return 0;
    }
}
```

Then open your terminal and try to download a youtube video:

```bash
$ php index.php dl https://www.youtube.com/watch?v=wtl5UrrgU8c

Video formats
1: video/mp4; codecs="avc1.640028" - 39013359
2: video/webm; codecs="vp9" - 35592706
3: video/mp4; codecs="av01.0.08M.08" - 33395130
4: video/mp4; codecs="avc1.4d401f" - 9282564
5: video/webm; codecs="vp9" - 11742153
6: video/mp4; codecs="av01.0.05M.08" - 19186717
7: video/mp4; codecs="avc1.4d401e" - 5689359
8: video/webm; codecs="vp9" - 6911794
9: video/mp4; codecs="av01.0.04M.08" - 8974711
10: video/mp4; codecs="avc1.4d401e" - 4401516
11: video/webm; codecs="vp9" - 4782315
12: video/mp4; codecs="av01.0.01M.08" - 5253856
13: video/mp4; codecs="avc1.4d4015" - 2686265
14: video/webm; codecs="vp9" - 3085549
15: video/mp4; codecs="av01.0.00M.08" - 2907767
16: video/mp4; codecs="avc1.4d400c" - 1378381
17: video/webm; codecs="vp9" - 2921774
18: video/mp4; codecs="av01.0.00M.08" - 2401547
19: audio/mp4; codecs="mp4a.40.2" - 4881930
20: audio/webm; codecs="opus" - 1962598
21: audio/webm; codecs="opus" - 2514707
22: audio/webm; codecs="opus" - 4880453
Which format do you want to download? (default: 1):10
Connected...
Mime-type: video/mp4
Filesize: 4401516
[====================================================================================================>] 100% ( 4/ 4 mb)
```


Cool, right?

### Download from a direct link

So far, we can download youtube videos, but what about direct links such as http://example.com/file.mp3?

Because we used the Factory design pattern and Single Responsibility principle, it would be as easy as create a separate class for direct links.

Create `Direct.php` file under `src/Providers` directory with the following contents:

```php
<?php

namespace Downloader\Providers;

use Downloader\ProviderAbstract;

class Direct extends ProviderAbstract
{

    public function init(string $url): void
    {
        $this->url = $url;
        $this->name = basename($url);
    }
}
```

Because direct links are _direct_, the only thing we should do is to set `$this->url` and `$this->name`. Now, open `src/Factory.php` and change it as below:

```php
<?php
// src/Factory.php

namespace Downloader;

use Downloader\Providers\Direct;
use Downloader\Providers\Youtube;

class Factory
{
    public static function make(string $url): ProviderAbstract
    {
        switch (true) {
            case preg_match('%(?:youtube(?:-nocookie)?\.com/(?:[^/]+/.+/|(?:v|e(?:mbed)?)/|.*[?&]v=)|youtu\.be/)([^"&?/ ]{11})%i', $url, $matches):
                return new Youtube($url);
            default:
                return new Direct($url);
        }
    }
}
```

As you can see, we changed the `default` section to return a new object of `Direct` class.

That's all we must do if we want to handle other sources. 


## Summary

Although PHP is not an excellent tool to write a downloader, we saw how we could manage and download a file from Youtube or a direct link. We learned about **Stream Context** and how we can use them of change or get information about a stream. We used the Factory design pattern and saw what the heck **Single Responsibility** means. Now it's your time to use your imagination and creativity to extend this project. Maybe by adding more providers or how to make use of multithreading extensions such as https://github.com/krakjoe/parallel to make it more usable.

