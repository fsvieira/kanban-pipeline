# kanban-pipeline

A processing pipeline based on kanban methodology.

# install
```
npm install kanban-pipeline
```

# Use

I wanted to present a real use case so this is the code for it:

```
/* 
  Import kanban and events,
  we use our custom events but there is also some events alredy defined and triggerd by kanban.
*/
const {Kanban, Events} = require("kanban-pipeline");

// Imports needed for this examples, not needed by kanban lib.
const fs = require("fs");
const path = require("path");
var mkdirp = require('mkdirp');

const events = new Events();

/*
  An helper function to list all folders and files on a directory (not recursive).
*/
function listDir (dirname) {
    return new Promise((resolve, reject) => {
        fs.readdir(dirname, (err, items) => {
            const stats = [];
            for (let i=0; i<items.length; i++) {
                const item = items[i];
                const itemPath = path.join(dirname, item);

                stats.push(
                    new Promise((resolve, reject) =>
                        fs.lstat(itemPath, (error, stats) => {
                                if (error) {
                                    reject(error);
                                }
                                else {
                                    resolve({
                                        name: item,
                                        path: itemPath,
                                        type: stats.isFile()?"FILE":"DIR"
                                    });
                                }
                            }
                        )
                    )
                );
            }

            Promise.all(stats).then(resolve, reject);
        });
    });
}

/*
  Here we define the pipeline, the pipeline contains the transitions states, all 
  states are executed async with a promise.
*/
const pipeline = {
    transitions: {
        /*
          Define a listDir transition
        */
        listDir: {
            /*
              process is the processing function of this transition.
              All process function receive two arguments:
                * req: request object, with information about the request.
                * res: the response object, to send information to next state.
            */
            process: (req, res) => {
                /*
                  The req object always contains a args field that holds the value 
                  passed from previews state transitions or by user invocation of state.
                  In this case we want to get path, filter and destination.
                */
                const {path, filter, destination} = req.args;

                // list the requested path directory:
                listDir(path)
                    // setup the response,
                    .then(items => items.map(
                        item => {
                            item.filter = filter;
                            item.destination = destination;

                            return item;
                        }
                    ))
                    /* 
                    send the response to next transtion, with res.send;
                    Here we are using {values: items}, this tells kanban to pass 
                    each item in items to next transition individually, if we 
                    want to pass the all array we would do {value: item}.
                    */
                    .then(items => res.send({values: items}))
                ;
            },
            // to: contains the next allowed states from this state transition.
            to: ["listDir", "filterFile"],
            /*
            Dispatch function routes the value to its correct transition state,
            dispatch is always executed after processing. If the "to" field only 
            contains one state we don't need to define any dispatch function.
            In this case dispatch function wiil send directories to listDir and 
            files to filterFile. By sending folders to listDir we are able to 
            reuse this state to walk the subfolders tree.
            */
            dispatch: value => value.type==="DIR"?"listDir":"filterFile"
        },
        filterFile: {
            process: (req, res) => {
                /*
                  On this state we know that we can only get files, because previews transition state 
                  as filtered files for us.
                */
                const file = req.args;

                // We want to find a specific file name,
                const params = file.name.match(/example_([0-9]+)-([0-9]+)-([0-9]+)_([^.]+).json/);
                if (params) {
                    const [filename, year, month, day] = params;

                    // extract information from filename,
                    file.description = {
                        filename,
                        year: +year,
                        month: +month,
                        day: +day
                    };

                    // check if this is the file we are locking for:
                    if (file.filter(file.description)) {
                        // send it to next state,
                        res.send({value: file});
                    }
                }
            },
            to: ["copyFile"]
            // because we only have a to state, we don't need dispatch function.
        },
        copyFile: {
            process: (req, res) => {
                /*
                  We want to organize files by year/month, so we are going to create directories and 
                  copy the file to their.
                */
                const {name: filename, path: filePath, destination, description: {year, month}} = req.args;
                const monthStr = (month < 10? "0": "") + month;

                const toDestinationDir = path.join(destination, ""+year, monthStr);

                mkdirp(toDestinationDir, error => {
                    if (error) {
                        events.trigger("error", err);
                        /*
                          Sending an empty object will terminate the processing,
                          kanban only process objects that contains value or values.
                        */
                        res.send({});
                    }
                    else {
                        const toDestinationFile = path.join(toDestinationDir, filename);

                        fs.copyFile(filePath, toDestinationFile, (error) => {
                            if (error) {
                                events.trigger("error", error);
                                
                            }
                            else {
                                events.trigger("result", "File " + filePath + " has been copied to " + toDestinationDir);
                            }
                            
                            /*
                              Sending an empty object will terminate the processing,
                              kanban only process objects that contains value or values.
                            */
                            res.send({});
                        });
                    }
                    
                });

                
            }
        }
    },
    /*
      We can order states, so they are processed in order. For example:
      ordered: ["listDir", "filterFile"]
      It would make kanban to process items in the order that they are inserted. This means 
      that kanban would wait for a process to end before starting to process another data on pipeline.
      In this case we don't have any order, so kanban executes every data on pipeline as soon as possible.
    */
    ordered: [],
    /*
    The processing start point, its mandatory. 
    */
    start: "listDir"
};

// Out custom events, they are being redirected to console.log
events.on("result", console.log);
events.on("error", err => console.log("Error: " + err));

// Init kanban with pipeline and events.
const search = new Kanban(pipeline, events);

const dirname = ".";

/*
  Starts the processing of a value, this will be added to start transition pipeline.
  trackId is used to track the value and its childs, kanban will trigger end event when 
  this value is no longer being processed, and also other events using trackId.
*/
search.add({value: {
    path: dirname,
    filter: ({year}) => [2015, 2016, 2017, 2018].indexOf(year) !== -1,
    destination: "test/"
}, trackId: dirname});

```

