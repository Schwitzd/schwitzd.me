+++
title = 'Not in My Picture'
date = 2024-05-10T11:39:18Z
draft = false
+++

Surely you have been in a situation where you wanted to wait before taking a photo because there were other people in the lens. Or think about how beautiful the photo of that square would have been without the people.

That's it... I do this almost every time I take a photo (apart from the time I was in the middle of nowhere in Iceland).
So I said to myself: either I get up at 5am and take the photo, hoping that nobody else has had my bright idea, or we try to remove the objects with AI.

TL;DR: I woke up at 5am.

## My take aways

If the object or person to be removed is in the foreground, there are no shadows and the background is not too complicated, it works almost perfectly, otherwise bye-bye!
Practically every photo where I have tried to remove one or more people has had a very poor result.

Of course, this is an ever-evolving technology and will only get better in the coming years... so it is worth learning how to use it. Especially since you can put it in a container :)

## IOPaint

### Getting started

While searching the internet I came across [IOPaint](https://github.com/Sanster/IOPaint) and decided to give it a try. Inside the repository you can find the dockerfile already, but I decided to [build mine](https://github.com/Schwitzd/docker-iopaint).

1. Clone this repository to your local machine:

    ```bash
    git clone https://github.com/Schwitzd/docker-iopaint.git
    ```

2. Navigate to the cloned directory:

    ```bash
    cd docker-iopaint
    ```

3. Build and run the Docker container using docker-compose:

    ```bash
    docker-compose up --build -d
    ```

4. Once the container is up and running, you can access IOPaint at `http://localhost:8080` in your web browser.

### How to use

I will not make a tutorial on how to use IOPaint, you can find the [official documentation](https://www.iopaint.com/), and on Youtube you can find lots of videos.

## Result

This was my original picture I shooted, the square was full of people goin back and forth and was impossible to take a picture of the abandoned hotel alone.

{{< figure src="https://onedrive.live.com/embed?resid=DC941554AACD227C%2112277&authkey=%21AACf8t5beAj9jCU&width=1024" align="center" caption="Original picture">}}

Taaac... the result after I removed the three people, as you can see, starting behind the shadow, the image is blurred and instead of creating the pillar behind the man's head, a mess happened.

{{< figure src="https://onedrive.live.com/embed?resid=DC941554AACD227C%2112276&authkey=%21AIkigMEf39hbLgg&width=1024" align="center" caption="AI processed picture">}}

This is just an example, I tried to process several images but the end result was always similar.
