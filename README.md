Note: while following this solution, type commands in a single shell session (or use a single script) unless said otherwise.
This is because it uses variables and file descriptors, they are not directly available outside their original session.
 
Create a temporary fifo. The right way to create a temporary file is with mktemp. Unfortunately it cannot create fifos.
It can create a temporary directory though:
 
tmpd=`mktemp -d`
tmpf="$tmpd"/fifo
mkfifo "$tmpf"
printf "%s\n" "$tmpf"  # just to know the path to the fifo, it may be useful later
 
Alternatively you can create a named fifo by hand in some fixed or ad-hoc location. It's up to you.
 
Create a background process that will read the fifo and pass data to a server:
 
nc 192.168.1.115 12345 < "$tmpf" &
ncpid=$!  # PID may be useful later
 
Pay attention if nc doesn't exit prematurely. Assuming there's no problem with the connection itself nor the server,
the above background command will stay connected until you finish sending the first bunch of data through the fifo.
But you want it to stay open and accept multiple writes, so open the fifo and don't close it (yet):
 
exec 3> "$tmpf"
 
Now you can send whatever you like through the fifo and the background connection persists:
 
echo abcd      >&3  # sends text
echo -e '\x80' >&3  # sends "binary"
cat /etc/issue >&3  # sends file
cat            >&3  # type whatever you want, terminate with Ctrl+D
 
Any of this commands can be invoked with the path to the fifo instead of &3, if only you know the path. Knowing the path,
you can write to the fifo from the same or another shell session:
 
cat > /path/to/the/fifo  # type whatever you want, terminate with Ctrl+D
 
Or you can open a descriptor in another session in a similar manner like in the original one.
Whatever way you write to the fifo, the nc will pass it to the remote server in a single connection.
 
But beware of race conditions. Using a single descriptor inside a single shell session is a good way to avoid them.
 
After you pass whatever data you need, terminate the nc and close the descriptor in the original shell session:
 
kill $ncpid
exec 3>&-
 
Note in some circumstances the sole latter command is enough to make the background nc exit;
it depends on what you did and what you are still doing with the fifo. For this reason I chose to kill the nc explicitly.
 
Remove the temporary directory and its content (i.e. the fifo):
 
rm -r "$tmpd"
 
Final note:
 
It's not necessary to put nc into background in the first place. You can run it in one terminal,
write to the fifo (knowing its path) from another one. This way you can easily monitor its state.
Tailor this solution to your needs.
 
 
Listen way:
 
You can get help from a named pipe to do it, with a little trick:
 
remote$ nc -v -n -l -p 8888 > /tmp/data
Listening on [0.0.0.0] (family 0, port 8888)
Connection from 10.0.3.1 47108 received!
 
local:1$ mkfifo /tmp/fifo
local:1$ nc -v -n <>/tmp/fifo 10.0.3.66 8888
Connection to 10.0.3.66 8888 port [tcp/*] succeeded!
 
It's important to open the fifo read-write (or else have an other neutral process like sleep 9999 >/tmp/fifo keeping it opened for writing) else the first (end of) write done will end netcat's read loop. Hence the <>.
 
local:2$ echo aaaaaaaaaabbbbbbbbbbccccccccccc > /tmp/fifo
local:2$ cat /bin/ls > /tmp/fifo # sends the binary contents of the file /bin/ls, which would probably garble the input or output
local:2$ cat > /tmp/fifo
Some more interactive text
 
(local:1$ )
^C
 
remote$ od -c /tmp/data|head -4
0000000   a   a   a   a   a   a   a   a   a   a   b   b   b   b   b   b
0000020   b   b   b   b   c   c   c   c   c   c   c   c   c   c   c  \n
0000040 177   E   L   F 002 001 001  \0  \0  \0  \0  \0  \0  \0  \0  \0
0000060 003  \0   >  \0 001  \0  \0  \0   0   T  \0  \0  \0  \0  \0  \0
