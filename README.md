
# resplit

Containerizable utility to read lines of text from input and push into a Redis list.

<img src="https://raw.githubusercontent.com/evanx/resplit/master/docs/readme/main.png"/>

## Use case

Say we need to import a text file, or lines of output from some process, for further processing.

If we wish to write simple Redis-driven microservices, then the first step is to stream those lines of text into Redis.

This utility can perform that first step. It can be built and run as a Docker container for convenience.

## Test run

See `development/run.sh` https://github.com/evanx/resplit/blob/master/development/run.sh

We will use the utility to `lpush` lines into the Redis key `test:resplit`
```
redisKey='test:resplit'
```
Let's delete the key for starters.
```
redis-cli del $redisKey
```
Now we `npm start` our test run:
```
(
echo 'line 1'
echo 'line 2'
echo 'line 3'
) | redisKey=$redisKey npm start
```
We inspect the list:
```
$ redis-cli lrange $redisKey 0 5
1) "line 3"
2) "line 2"
3) "line 1"
```
where we notice that the lines are in reverse order. This is because the utility performs an `lpush` (left push) operation, and so the last line is at the head (left side) of the list.

See https://redis.io/commands/lpush

Therefore for further processing of the imported lines in order, we would use `RPOP` to pop lines from right side (tail) of the list:
```
$ redis-cli rpop $redisKey
"line 1"
```

## Config spec

See `lib/spec.js` https://github.com/evanx/resplit/blob/master/lib/spec.js
```javascript
module.exports = {
    description: 'Containerizable utility to read lines of text from input and push into a Redis list.',
    required: {
        redisHost: {
            description: 'the Redis host',
            default: 'localhost'
        },
        redisPort: {
            description: 'the Redis port',
            default: 6379
        },
        redisPassword: {
            description: 'the Redis password',
            required: false
        },
        redisKey: {
            description: 'the Redis list key'
        },
        highLength: {
            description: 'the length of the list for back-pressure',
            default: 500
        },
        delayMillis: {
            description: 'the delay duration in milliseconds when back-pressure',
            unit: 'ms',
            default: 5000
        },
        loggerLevel: {
            description: 'the logging level',
            default: 'info',
            example: 'debug'
        }
    }
};
```
where `redisKey` is the list to which the utility will `lpush` the lines from standard input.

## Implementation

See `lib/main.js` https://github.com/evanx/resplit/blob/master/lib/main.js
```javascript
inputStream.pipe(split())
.on('error', err => reject(err))
.on('data', line => client.lpushAsync(config.redisKey, line)
.catch(err => reject(err)))
.on('end', () => resolve());
```

Incidently we delay the input stream using the length of the Redis list for back-pressure:
```javascript
const inputStreamTransform = function(buf, enc, next) {
    this.push(buf);
    client.llen(config.redisKey, (err, llen) => {
        if (err) {
            this.emit('error', err);
        } else if (llen > config.highLength) {
            logger.warn({llen}, config.delayMillis);
            promiseDelay(config.delayMillis).then(next);
        } else {
            next();
        }
    })
};
```
where we delay calling `next()` via `promiseDelay` when the length of the Redis list is somewhat high.

### Appication archetype

Incidently `lib/index.js` uses the `redis-util-app-rpf` application archetype.
```
require('./redis-util-app-rpf')(require('./spec'), require('./main'));
```
where we extract the `config` from `process.env` according to the `spec` and invoke our `main` function.

That archetype is embedded in the project, as it is still evolving. Also, you can find it at https://github.com/evanx/redis-util-app-rpf.

This provides lifecycle boilerplate to reuse across similar applications.

## Docker

You can build as follows:
```
docker build -t resplit https://github.com/evanx/resplit.git
```
from https://github.com/evanx/resplit/blob/master/Dockerfile

See `test/demo.sh` https://github.com/evanx/resplit/blob/master/test/demo.sh
```
cat test/lines.txt |
  docker run \
  --network=resplit-network \
  --name resplit-app \
  -e NODE_ENV=production \
  -e redisHost=$encipherHost \
  -e redisPort=$encipherPort \
  -e redisPassword=$redisPassword \
  -e redisKey=$redisKey \
  -i evanxsummers/resplit
```
having:
- isolated network `resplit-network`
- isolated Redis instance named `resplit-redis` using image `tutum/redis`
- a pair of `spiped` containers for encrypt/decrypt tunnelling
- the prebuilt image `evanxsummers/resplit` used in interactive mode via `-i`

### Redis container

The demo uses `tutum/redis` where we use `docker logs` to get the password:
```
docker logs $redisContainer | grep '^\s*redis-cli -a'
```

### spiped

See https://github.com/Tarsnap/spiped

We generate a `keyfile` as follows
```
dd if=/dev/urandom bs=32 count=1 > $HOME/tmp/test-spiped-keyfile
```

We then create the two ends of the tunnel using the `keyfile`:
```
decipherContainer=`docker run --network=resplit-network \
  --name resplit-decipher -v $HOME/tmp/test-spiped-keyfile:/spiped/key:ro \
  -p 6444:6444 -d spiped \
  -d -s "[0.0.0.0]:6444" -t "[$redisHost]:6379"`
```
```
encipherContainer=`docker run --network=resplit-network \
  --name resplit-encipher -v $HOME/tmp/test-spiped-keyfile:/spiped/key:ro \
  -p $encipherPort:$encipherPort -d spiped \
  -e -s "[0.0.0.0]:$encipherPort" -t "[$decipherHost]:6444"`
```

### Tear down

We remove the test images:
```
docker rm -f resplit-redis resplit-app resplit-decipher resplit-encipher
```
Finally we remove the test network:
```
docker network rm resplit-network
```

<hr>
https://twitter.com/@evanxsummers
