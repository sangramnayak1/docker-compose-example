<body class="colums">
  <p>
    This tutorial is designed to introduce the key concepts of Docker Compose whilst building a simple Python web application. The application uses the Flask framework and maintains a hit counter in
    Redis.
  </p>

  <p>The concepts demonstrated here should be understandable even if you’re not familiar with Python.</p>

  <h2 id="prerequisites">Prerequisites</h2>

  <p>You need to have Docker Engine and Docker Compose on your machine. You can either:</p>
  <ul>
    <li>Install <a href="/get-docker/">Docker Engine</a> and <a href="/compose/install/">Docker Compose</a> as standalone binaries</li>
    <li>Install <a href="/desktop/">Docker Desktop</a> which includes both Docker Engine and Docker Compose</li>
  </ul>

  <p>You don’t need to install Python or Redis, as both are provided by Docker images.</p>

  <h2 id="step-1-define-the-application-dependencies">Step 1: Define the application dependencies</h2>

  <ol>
    <li>
      <p>Create a directory for the project:</p>

      <div class="language-console highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span><span class="nb">mkdir </span>composetest
<span class="gp">$</span><span class="w"> </span><span class="nb">cd </span>composetest
</code></pre>
        </div>
      </div>
    </li>
    <li>
      <p>Create a file called <code class="language-plaintext highlighter-rouge">app.py</code> in your project directory and paste the following code in:</p>

      <div class="language-python highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code><span class="kn">import</span> <span class="nn">time</span>

<span class="kn">import</span> <span class="nn">redis</span>
<span class="kn">from</span> <span class="nn">flask</span> <span class="kn">import</span> <span class="n">Flask</span>

<span class="n">app</span> <span class="o">=</span> <span class="n">Flask</span><span class="p">(</span><span class="n">__name__</span><span class="p">)</span>
<span class="n">cache</span> <span class="o">=</span> <span class="n">redis</span><span class="p">.</span><span class="n">Redis</span><span class="p">(</span><span class="n">host</span><span class="o">=</span><span class="s">'redis'</span><span class="p">,</span> <span class="n">port</span><span class="o">=</span><span class="mi">6379</span><span class="p">)</span>

<span class="k">def</span> <span class="nf">get_hit_count</span><span class="p">():</span>
    <span class="n">retries</span> <span class="o">=</span> <span class="mi">5</span>
    <span class="k">while</span> <span class="bp">True</span><span class="p">:</span>
        <span class="k">try</span><span class="p">:</span>
            <span class="k">return</span> <span class="n">cache</span><span class="p">.</span><span class="n">incr</span><span class="p">(</span><span class="s">'hits'</span><span class="p">)</span>
        <span class="k">except</span> <span class="n">redis</span><span class="p">.</span><span class="n">exceptions</span><span class="p">.</span><span class="nb">ConnectionError</span> <span class="k">as</span> <span class="n">exc</span><span class="p">:</span>
            <span class="k">if</span> <span class="n">retries</span> <span class="o">==</span> <span class="mi">0</span><span class="p">:</span>
                <span class="k">raise</span> <span class="n">exc</span>
            <span class="n">retries</span> <span class="o">-=</span> <span class="mi">1</span>
            <span class="n">time</span><span class="p">.</span><span class="n">sleep</span><span class="p">(</span><span class="mf">0.5</span><span class="p">)</span>

<span class="o">@</span><span class="n">app</span><span class="p">.</span><span class="n">route</span><span class="p">(</span><span class="s">'/'</span><span class="p">)</span>
<span class="k">def</span> <span class="nf">hello</span><span class="p">():</span>
    <span class="n">count</span> <span class="o">=</span> <span class="n">get_hit_count</span><span class="p">()</span>
    <span class="k">return</span> <span class="s">'Hello World! I have been seen {} times.</span><span class="se">\n</span><span class="s">'</span><span class="p">.</span><span class="nb">format</span><span class="p">(</span><span class="n">count</span><span class="p">)</span>
</code></pre>
        </div>
      </div>

      <p>In this example, <code class="language-plaintext highlighter-rouge">redis</code> is the hostname of the redis container on the
        application’s network. We use the default port for Redis, <code class="language-plaintext highlighter-rouge">6379</code>.</p>

      <blockquote>
        <p>Handling transient errors</p>

        <p>Note the way the <code class="language-plaintext highlighter-rouge">get_hit_count</code> function is written. This basic retry
          loop lets us attempt our request multiple times if the redis service is
          not available. This is useful at startup while the application comes
          online, but also makes the application more resilient if the Redis
          service needs to be restarted anytime during the app’s lifetime. In a
          cluster, this also helps handling momentary connection drops between
          nodes.</p>
      </blockquote>
    </li>
    <li>
      <p>Create another file called <code class="language-plaintext highlighter-rouge">requirements.txt</code> in your project directory and
        paste the following code in:</p>

      <div class="language-text highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code>flask
redis
</code></pre>
        </div>
      </div>
    </li>
  </ol>

  <h2 id="step-2-create-a-dockerfile">Step 2: Create a Dockerfile</h2>

  <p>The Dockerfile is used to build a Docker image. The image
    contains all the dependencies the Python application requires, including Python
    itself.</p>

  <p>In your project directory, create a file named <code class="language-plaintext highlighter-rouge">Dockerfile</code> and paste the following code in:</p>

  <div class="language-dockerfile highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="c"># syntax=docker/dockerfile:1</span>
<span class="k">FROM</span><span class="s"> python:3.7-alpine</span>
<span class="k">WORKDIR</span><span class="s"> /code</span>
<span class="k">ENV</span><span class="s"> FLASK_APP=app.py</span>
<span class="k">ENV</span><span class="s"> FLASK_RUN_HOST=0.0.0.0</span>
<span class="k">RUN </span>apk add <span class="nt">--no-cache</span> gcc musl-dev linux-headers
<span class="k">COPY</span><span class="s"> requirements.txt requirements.txt</span>
<span class="k">RUN </span>pip <span class="nb">install</span> <span class="nt">-r</span> requirements.txt
<span class="k">EXPOSE</span><span class="s"> 5000</span>
<span class="k">COPY</span><span class="s"> . .</span>
<span class="k">CMD</span><span class="s"> ["flask", "run"]</span>
</code></pre>
    </div>
  </div>

  <p>This tells Docker to:</p>

  <ul>
    <li>Build an image starting with the Python 3.7 image.</li>
    <li>Set the working directory to <code class="language-plaintext highlighter-rouge">/code</code>.</li>
    <li>Set environment variables used by the <code class="language-plaintext highlighter-rouge">flask</code> command.</li>
    <li>Install gcc and other dependencies</li>
    <li>Copy <code class="language-plaintext highlighter-rouge">requirements.txt</code> and install the Python dependencies.</li>
    <li>Add metadata to the image to describe that the container is listening on port 5000</li>
    <li>Copy the current directory <code class="language-plaintext highlighter-rouge">.</code> in the project to the workdir <code class="language-plaintext highlighter-rouge">.</code> in the image.</li>
    <li>Set the default command for the container to <code class="language-plaintext highlighter-rouge">flask run</code>.</li>
  </ul>

  <blockquote class="important">
    <p>Important</p>

    <p>Check that the <code class="language-plaintext highlighter-rouge">Dockerfile</code> has no file extension like <code class="language-plaintext highlighter-rouge">.txt</code>. Some editors may append this file extension automatically which results in an error when you run the application.</p>
  </blockquote>

  <p>For more information on how to write Dockerfiles, see the
    <a href="/develop/">Docker user guide</a>
    and the <a href="/engine/reference/builder/">Dockerfile reference</a>.
  </p>

  <h2 id="step-3-define-services-in-a-compose-file">Step 3: Define services in a Compose file</h2>

  <p>Create a file called <code class="language-plaintext highlighter-rouge">compose.yaml</code> in your project directory and paste
    the following:</p>

  <div class="language-yaml highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="na">services</span><span class="pi">:</span>
  <span class="na">web</span><span class="pi">:</span>
    <span class="na">build</span><span class="pi">:</span> <span class="s">.</span>
    <span class="na">ports</span><span class="pi">:</span>
      <span class="pi">-</span> <span class="s2">"</span><span class="s">8000:5000"</span>
  <span class="na">redis</span><span class="pi">:</span>
    <span class="na">image</span><span class="pi">:</span> <span class="s2">"</span><span class="s">redis:alpine"</span>
</code></pre>
    </div>
  </div>

  <p>This Compose file defines two services: <code class="language-plaintext highlighter-rouge">web</code> and <code class="language-plaintext highlighter-rouge">redis</code>.</p>

  <p>The <code class="language-plaintext highlighter-rouge">web</code> service uses an image that’s built from the <code class="language-plaintext highlighter-rouge">Dockerfile</code> in the current directory.
    It then binds the container and the host machine to the exposed port, <code class="language-plaintext highlighter-rouge">8000</code>. This example service uses the default port for the Flask web server, <code class="language-plaintext highlighter-rouge">5000</code>.</p>

  <p>The <code class="language-plaintext highlighter-rouge">redis</code> service uses a public <a href="https://registry.hub.docker.com/_/redis/">Redis</a>
    image pulled from the Docker Hub registry.</p>

  <h2 id="step-4-build-and-run-your-app-with-compose">Step 4: Build and run your app with Compose</h2>

  <ol>
    <li>
      <p>From your project directory, start up your application by running <code class="language-plaintext highlighter-rouge">docker compose up</code>.</p>

      ```
      docker compose up

      Creating network "composetest_default" with the default driver
      Creating composetest_web_1 ...
      Creating composetest_redis_1 ...
      Creating composetest_web_1
      Creating composetest_redis_1 ... done
      Attaching to composetest_web_1, composetest_redis_1
      web_1 | * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
      redis_1 | 1:C 17 Aug 22:11:10.480 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
      redis_1 | 1:C 17 Aug 22:11:10.480 # Redis version=4.0.1, bits=64, commit=00000000, modified=0, pid=1, just started
      redis_1 | 1:C 17 Aug 22:11:10.480 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
      web_1 | * Restarting with stat
      redis_1 | 1:M 17 Aug 22:11:10.483 * Running mode=standalone, port=6379.
      redis_1 | 1:M 17 Aug 22:11:10.483 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
      web_1 | * Debugger is active!
      redis_1 | 1:M 17 Aug 22:11:10.483 # Server initialized
      redis_1 | 1:M 17 Aug 22:11:10.483 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
      web_1 | * Debugger PIN: 330-787-903
      redis_1 | 1:M 17 Aug 22:11:10.483 * Ready to accept connections
      ```

      <p>
        Compose pulls a Redis image, builds an image for your code, and starts the
        services you defined. In this case, the code is statically copied into the image at build time.
      </p>
    </li>

    <li>
      <p>Enter http://localhost:8000/ in a browser to see the application running.</p>

      <p>If this doesn’t resolve, you can also try http://127.0.0.1:8000.</p>

      <p>You should see a message in your browser saying:</p>

      <div class="language-console highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code><span class="go">Hello World! I have been seen 1 times.
</span></code></pre>
        </div>
      </div>

      <p><img src="/compose/images/quick-hello-world-1.png" alt="hello world in browser" /></p>
    </li>

    <li>
      <p>Refresh the page.</p>

      <p>The number should increment.</p>

      <div class="language-console highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code><span class="go">Hello World! I have been seen 2 times.
</span></code></pre>
        </div>
      </div>
      <p><img src="/compose/images/quick-hello-world-2.png" alt="hello world in browser" /></p>
    </li>

    <li>
      <p>Switch to another terminal window, and type <code class="language-plaintext highlighter-rouge">docker image ls</code> to list local images.</p>

      <p>Listing images at this point should return <code class="language-plaintext highlighter-rouge">redis</code> and <code class="language-plaintext highlighter-rouge">web</code>.</p>

      <div class="language-console highlighter-rouge">
        <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>docker image <span class="nb">ls</span>
<span class="go">
REPOSITORY        TAG           IMAGE ID      CREATED        SIZE
composetest_web   latest        e2c21aa48cc1  4 minutes ago  93.8MB
python            3.4-alpine    84e6077c7ab6  7 days ago     82.5MB
redis             alpine        9d8fa9aa0e5b  3 weeks ago    27.5MB
</span></code></pre>
        </div>
      </div>

      <p>You can inspect images with <code class="language-plaintext highlighter-rouge">docker inspect &lt;tag or id&gt;</code>.</p>
    </li>

    <li>
      <p>Stop the application, either by running <code class="language-plaintext highlighter-rouge">docker compose down</code>
        from within your project directory in the second terminal, or by
        hitting CTRL+C in the original terminal where you started the app.</p>
    </li>

  </ol>

  <h2 id="step-5-edit-the-compose-file-to-add-a-bind-mount">Step 5: Edit the Compose file to add a bind mount</h2>

  <p>Edit the <code class="language-plaintext highlighter-rouge">compose.yaml</code> file in your project directory to add a
    <a href="/storage/bind-mounts/">bind mount</a> for the <code class="language-plaintext highlighter-rouge">web</code> service:
  </p>

  <div class="language-yaml highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="na">services</span><span class="pi">:</span>
  <span class="na">web</span><span class="pi">:</span>
    <span class="na">build</span><span class="pi">:</span> <span class="s">.</span>
    <span class="na">ports</span><span class="pi">:</span>
      <span class="pi">-</span> <span class="s2">"</span><span class="s">8000:5000"</span>
    <span class="na">volumes</span><span class="pi">:</span>
      <span class="pi">-</span> <span class="s">.:/code</span>
    <span class="na">environment</span><span class="pi">:</span>
      <span class="na">FLASK_DEBUG</span><span class="pi">:</span> <span class="s2">"</span><span class="s">true"</span>
  <span class="na">redis</span><span class="pi">:</span>
    <span class="na">image</span><span class="pi">:</span> <span class="s2">"</span><span class="s">redis:alpine"</span>
</code></pre>
    </div>
  </div>

  <p>The new <code class="language-plaintext highlighter-rouge">volumes</code> key mounts the project directory (current directory) on the
    host to <code class="language-plaintext highlighter-rouge">/code</code> inside the container, allowing you to modify the code on the
    fly, without having to rebuild the image. The <code class="language-plaintext highlighter-rouge">environment</code> key sets the
    <code class="language-plaintext highlighter-rouge">FLASK_DEBUG</code> environment variable, which tells <code class="language-plaintext highlighter-rouge">flask run</code> to run in development
    mode and reload the code on change. This mode should only be used in development.
  </p>

  <h2 id="step-6-re-build-and-run-the-app-with-compose">Step 6: Re-build and run the app with Compose</h2>

  <p>From your project directory, type <code class="language-plaintext highlighter-rouge">docker compose up</code> to build the app with the updated Compose file, and run it.</p>

  <div class="language-console highlighter-rouge">
    <div class="highlight">
      <pre class="highlight">
		<code>
			<span class="gp">$</span><span class="w"></span>docker compose up
			<span class="go">
Creating network "composetest_default" with the default driver
Creating composetest_web_1 ...
Creating composetest_redis_1 ...
Creating composetest_web_1
Creating composetest_redis_1 ... done
Attaching to composetest_web_1, composetest_redis_1
web_1    |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
			</span>
<span class="c">...</span></code></pre>
    </div>
  </div>

  <p>Check the <code class="language-plaintext highlighter-rouge">Hello World</code> message in a web browser again, and refresh to see the
    count increment.</p>

  <blockquote class="important">
    <p>Shared folders, volumes, and bind mounts</p>

    <ul>
      <li>
        <p>If your project is outside of the <code class="language-plaintext highlighter-rouge">Users</code> directory (<code class="language-plaintext highlighter-rouge">cd ~</code>), then you
          need to share the drive or location of the Dockerfile and volume you are using.
          If you get runtime errors indicating an application file is not found, a volume
          mount is denied, or a service cannot start, try enabling file or drive sharing.
          Volume mounting requires shared drives for projects that live outside of
          <code class="language-plaintext highlighter-rouge">C:\Users</code> (Windows) or <code class="language-plaintext highlighter-rouge">/Users</code> (Mac), and is required for <em>any</em> project on
          Docker Desktop for Mac and Docker Desktop for Windows that uses
          <a href="/desktop/faqs/windowsfaqs/#how-do-i-switch-between-windows-and-linux-containers">Linux containers</a>.
          For more information, see
          <a href="/desktop/settings/mac/#file-sharing">File sharing on Docker for Mac</a>,
          <a href="/desktop/settings/windows/#file-sharing">File sharing on Docker for Windows</a>, <a href="/desktop/settings/linux/#file-sharing">File sharing on Docker for Linux</a>.
          and the general examples on how to
          <a href="/storage/volumes/">Manage data in containers</a>.
        </p>
      </li>
      <li>
        <p>If you are using Oracle VirtualBox on an older Windows OS, you might encounter an issue with shared folders as described in this <a href="https://www.virtualbox.org/ticket/14920">VB trouble
            ticket</a>. Newer Windows systems meet the
          requirements for <a href="/desktop/install/windows-install/">Docker Desktop for Windows</a> and do not
          need VirtualBox.</p>
      </li>
    </ul>
  </blockquote>

  <h2 id="step-7-update-the-application">Step 7: Update the application</h2>

  <p>As the application code is now mounted into the container using a volume,
    you can make changes to its code and see the changes instantly, without having
    to rebuild the image.</p>

  <p>Change the greeting in <code class="language-plaintext highlighter-rouge">app.py</code> and save it. For example, change the <code class="language-plaintext highlighter-rouge">Hello World!</code>
    message to <code class="language-plaintext highlighter-rouge">Hello from Docker!</code>:</p>

  <div class="language-python highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="k">return</span> <span class="s">'Hello from Docker! I have been seen {} times.</span><span class="se">\n</span><span class="s">'</span><span class="p">.</span><span class="nb">format</span><span class="p">(</span><span class="n">count</span><span class="p">)</span>
</code></pre>
    </div>
  </div>

  <p>Refresh the app in your browser. The greeting should be updated, and the
    counter should still be incrementing.</p>

  <p><img src="/compose/images/quick-hello-world-3.png" alt="hello world in browser" /></p>

  <h2 id="step-8-experiment-with-some-other-commands">Step 8: Experiment with some other commands</h2>

  <p>If you want to run your services in the background, you can pass the <code class="language-plaintext highlighter-rouge">-d</code> flag
    (for “detached” mode) to <code class="language-plaintext highlighter-rouge">docker compose up</code> and use <code class="language-plaintext highlighter-rouge">docker compose ps</code> to
    see what is currently running:</p>

  <div class="language-console highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>docker compose up <span class="nt">-d</span>
<span class="go">
Starting composetest_redis_1...
Starting composetest_web_1...

</span><span class="gp">$</span><span class="w"> </span>docker compose ps
<span class="go">
       Name                      Command               State           Ports         
-------------------------------------------------------------------------------------
composetest_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp              
</span><span class="gp">composetest_web_1     flask run                        Up      0.0.0.0:8000-&gt;</span>5000/tcp
</code></pre>
    </div>
  </div>

  <p>The <code class="language-plaintext highlighter-rouge">docker compose run</code> command allows you to run one-off commands for your
    services. For example, to see what environment variables are available to the
    <code class="language-plaintext highlighter-rouge">web</code> service:
  </p>

  <div class="language-console highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>docker compose run web <span class="nb">env</span>
</code></pre>
    </div>
  </div>

  <p>See <code class="language-plaintext highlighter-rouge">docker compose --help</code> to see other available commands.</p>

  <p>If you started Compose with <code class="language-plaintext highlighter-rouge">docker compose up -d</code>, stop
    your services once you’ve finished with them:</p>

  <div class="language-console highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>docker compose stop
</code></pre>
    </div>
  </div>

  <p>You can bring everything down, removing the containers entirely, with the <code class="language-plaintext highlighter-rouge">down</code>
    command. Pass <code class="language-plaintext highlighter-rouge">--volumes</code> to also remove the data volume used by the Redis
    container:</p>

  <div class="language-console highlighter-rouge">
    <div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>docker compose down <span class="nt">--volumes</span>
</code></pre>
    </div>
  </div>

  <h2 id="where-to-go-next">Where to go next</h2>

  <ul>
    <li>Try the <a href="https://github.com/docker/awesome-compose">Sample apps with Compose</a></li>
    <li><a href="/compose/reference/">Explore the full list of Compose commands</a></li>
    <li><a href="/compose/compose-file/">Explore the Compose file reference</a></li>
    <li>To learn more about volumes and bind mounts, see <a href="/storage/">Manage data in Docker</a></li>
  </ul>

  <div id="side-toc-title">Contents:</div>
  <div id="img-modal1" class="modal1">
    <span class="close1" id="img-modal-close1">&times;</span>
    <img src="/" alt="/" class="modal-content1" id="img-modal-img1">
    <div id="img-modal-caption1"></div>
  </div>

  <ul id="my_toc" class="inline_toc">
    <li><a href="#prerequisites" class="nomunge">Prerequisites</a></li>
    <li><a href="#step-1-define-the-application-dependencies" class="nomunge">Step 1: Define the application dependencies</a></li>
    <li><a href="#step-2-create-a-dockerfile" class="nomunge">Step 2: Create a Dockerfile</a></li>
    <li><a href="#step-3-define-services-in-a-compose-file" class="nomunge">Step 3: Define services in a Compose file</a></li>
    <li><a href="#step-4-build-and-run-your-app-with-compose" class="nomunge">Step 4: Build and run your app with Compose</a></li>
    <li><a href="#step-5-edit-the-compose-file-to-add-a-bind-mount" class="nomunge">Step 5: Edit the Compose file to add a bind mount</a></li>
    <li><a href="#step-6-re-build-and-run-the-app-with-compose" class="nomunge">Step 6: Re-build and run the app with Compose</a></li>
    <li><a href="#step-7-update-the-application" class="nomunge">Step 7: Update the application</a></li>
    <li><a href="#step-8-experiment-with-some-other-commands" class="nomunge">Step 8: Experiment with some other commands</a></li>
    <li><a href="#where-to-go-next" class="nomunge">Where to go next</a></li>
  </ul>

</body>
