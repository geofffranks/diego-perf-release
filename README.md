# BOSH Diego Performance Release

This is a release to measure the performance of Diego. See the proposal [here](https://github.com/pivotal-cf-experimental/diego-dev-notes/blob/master/proposals/measuring_performance.md).

## Usage

### Prerequisites

Deploy diego-release, cf-release.  To deploy this release, create a BOSH
deployment manifest with as many pusher instances as you want to use for
testing.

### Running Fezzik

1. `bosh ssh stress_tests 0`
1. Run `/var/vcap/jobs/caddy/bin/1_fezzik` multiple times.
1. Output is stored in `/var/vcap/packages/fezzik/src/github.com/cloudfoundry-incubator/fezzik/reports.json`


### Running Cedar

#### Running Cedar locally

1. Target a default diego enabled CF deployment
1. Target a chosen org and space
1. cd src/code.cloudfoundry.org/diego-stress-tests/cedar/assets/stress-app
1. Precompile the stress-app to `assets/temp-app` by running `GOOS=linux GOARCH=amd64 go build -o ../temp-app/stress-app`
1. cd back to src/code.cloudfoundry.org/diego-stress-tests/cedar
1. Build the cedar binary with `go build`
1. Run the following to start a test
```bash
./cedar -n <number_of_batches> -k <max_in_flight> -domain <your-app-domain> [-tolerance <tolerance-factor>]
```

You can always run `./cedar -h` to see the list of options you can provide to
cedar. Worth noting one of Cedar's options is a config file that provides the
manifest paths for the different apps being pushed. The default `config.json`
can be found
[here](https://github.com/cloudfoundry/diego-stress-tests/blob/master/cedar/config.json)

#### Running Cedar from a BOSH deployment

1. Run `./scripts/generate-deployment-manifest` and deploy `diego-perf-release`
   with the generated manifest. If on bosh-lite, you can use
   `./scripts/generate-bosh-lite-manifests`.
1. Run `bosh ssh` to SSH to the `cedar` VM in the `cf-warden-diego-perf` deployment.
1. Run `sudo su`.
1. Run the following commands:
   ```bash
   # put the CF CLI on the PATH
   export PATH=/var/vcap/packages/cf-cli/bin:$PATH

   # target CF and create an org and space for the apps
   cf api api.bosh-lite.com --skip-ssl-validation
   cf auth admin admin
   cf create-org o
   cf create-space cedar -o o
   cf target -o o -s cedar

   cd /var/vcap/packages/cedar

   /var/vcap/packages/cedar/bin/cedar \
     -n 1 \
     -k 2 \
     -payload /var/vcap/packages/cedar/assets/temp-app \
     -config /var/vcap/packages/cedar/config.json \
     -domain bosh-lite.com \
     &
   ```

#### Automatically Running 10 batches of Cedar (Preferred)

The steps mentioned in the previous section are automated by the
`./cedar_script`. The script will push 10 batches of apps each in its own
spaces. Details below on how to run it:

1. Run `cd /var/vcap/jobs/cedar/bin`
1. Run the following command to run the experiment:
  ```
  ./cedar_script
  ```

  To delete the spaces from a previous experiment before running the experiment, run the script as:
  ```
  DELETE_SPACES="yes" ./cedar_script
  ```

  To resume the experiment from the `n`th batch (where `n` is a number from `1`
  to `10`), add `n` as an argument to the script. For example, to run from the
  fourth batch:
  ```
  ./cedar_script 4
  ```

To see the results of the experiment, see the file
`/var/vcap/sys/log/arborist/arborist-output.json`. The json file should list
all the apps pushed and started and whether they were pushed/started
successfully or not.

The script will also output the min/max timestamp for each batch in
`/var/vcap/data/cedar/min-<batch#>.json` and
`/var/vcap/data/cedar/max-<batch#>.json`.

### Aggregating results

#### Preprocessing using perfchug

`perfchug` is a tool that ships with the diego-perf-release. It takes log
output from cedar, bbs and auctioneer, processes it, and converts it into
something that can be fed into InfluxDB.

To use `perfchug` locally:

1. `cd <path>/diego-perf-release/src/code.cloudfoundry.org/diego-stress-tests/perfchug`
1. Run `go install` to build the executable
1. Move the executable into your `$PATH`

Once it's in your path, you can use `perfchug` by piping log output into it.

For example:

```
cat /var/vcap/sys/log/cedar/cedar.stdout.log | perfchug
```

will spit influxdb-friendly metrics to stdout.

#### Automatic downloading and aggregation

We wrote a script to automate the entire process. The script does the following:

1. Download brain, bbs & cedar job logs using bosh
1. Reduce the logs to the start/end timestamps of the experiments ran
1. Merge the logs from all jobs together
1. Run perfchug on the resulting log file
1. Insert the output of perfchug into influxdb
1. Run a fixed set of queries to get percentiles of requests latency among other interesting metrics

In order to use the script, you need to do the following:

1. You are on a jump box inside the deployment, e.g. director
1. You are bosh targeted to the right environment
1. You have perfchug, veritas and bosh on your PATH
1. Create a new directory and `cd` into it. This will be used as the working
   directory for the script. Bosh logs will be downloaded in this directory.
1. From that directory run:
```/path/to/cedar_results.sh http://url.to.influxdb:8086 <cedar_data_directory> /path/to/diego/manifest /path/to/perf/manifest [/path/to/output/file]```

`cedar_data_directory` is the directory containing the min/max timestamps of
each batch, most probably `/var/vcap/data/cedar`.

The output file will contain one line per query. All query results are valid
json. If there are no data points in InfluxDB, e.g. no failures, InfluxDB will
result an empty result, e.g. `{"results":[]}`

## Development

These tests are meant to be run against a real IaaS. However, it is possible to
run them against BOSH-Lite during development. A deployment manifest template is
in `templates/bosh-lite.yml`. Use
[spiff](https://github.com/cloudfoundry-incubator/spiff) to merge it with a
`director_uuid` stub.
