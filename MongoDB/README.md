# MongoDB

## PHP-MongoDB

### Create Connection
```php
public function createConnection()
    {
        try {
            $host = env('MONGO_DB_HOST');
            $database = env('MONGO_DB_DATABASE');
            $username = env('MONGO_DB_USERNAME');
            $password = env('MONGO_DB_PASSWORD');

            $access = "mongodb://" . $username . ":" . $password . "@" . $host . ":27017/" . $database . "?authMechanism=SCRAM-SHA-1";
            $conn = new \MongoDB\Client($access);

            return $conn;
        } catch (\Exception $e) {
            return $e->getMessage();
        }
    }
```    

### Create Bulk Connection

```php
    public function createBulkConnection()
    {
        try {
            $host = env('MONGO_DB_HOST');
            $database = env('MONGO_DB_DATABASE');
            $username = env('MONGO_DB_USERNAME');
            $password = env('MONGO_DB_PASSWORD');

            $access = "mongodb://" . $username . ":" . $password . "@" . $host . ":27017/" . $database . "?authMechanism=SCRAM-SHA-1";
            $manager = new \MongoDB\Driver\Manager($access);
            $writeConcern = new \MongoDB\Driver\WriteConcern(\MongoDB\Driver\WriteConcern::MAJORITY, 1000);
            $bulk = new \MongoDB\Driver\BulkWrite(['ordered' => true]);

            return [$manager, $writeConcern, $bulk];
        } catch (\Exception $e) {
            return $e->getMessage();
        }
    }
```

### Get Date from TimeStamp

```php
public function getDateFromTimestamp($utc_datetime)
    {
        $timestamp = json_decode($utc_datetime);
        $utcdatetime = new \MongoDB\BSON\UTCDateTime($timestamp);
        $datetime = $utcdatetime->toDateTime();
        $time = $datetime->format(DATE_RSS);
        $datetime = date("Y-m-d H:i:s", strtotime($time));
        return $datetime;
    }
```

### Get Distinct Number
```php
public function getRevisionNumbers($from_date, $to_date,$revision, $collection_name)
    {
        try {
            $from_date = new \MongoDB\BSON\UTCDateTime(strtotime($from_date) * 1000);
            $to_date = new \MongoDB\BSON\UTCDateTime(strtotime($to_date) * 1000);
            $conn = self::createConnection();
            $db = $conn->selectDatabase(env('MONGO_DB_DATABASE'));
            $collection = $db->selectCollection($collection_name);

            $response = $collection->distinct(
                $revision,
                [
                    'date' => [
                        '$gte' => $from_date,
                        '$lte' => $to_date
                    ],
                ]
            );

            return $response;
        } catch (\Exception $e) {
            throw new \Exception($e->getMessage());
        }
    }
```
### Get Max Value

```php
public function getMaxRevision($from_date, $to_date, $collection_name)
    {
        try {
            $conn = self::createConnection();
            $db = $conn->selectDatabase(env('MONGO_DB_DATABASE'));
            $collection = $db->selectCollection($collection_name);

            $from_date = new \MongoDB\BSON\UTCDateTime(strtotime($from_date) * 1000);
            $to_date = new \MongoDB\BSON\UTCDateTime(strtotime($to_date) * 1000);

            $query = [
                [
                    '$match' => [
                        'date' => [
                            '$gte' => $from_date,
                            '$lte' => $to_date
                        ],
                    ]
                ],
                [
                    '$group' => [
                        '_id' => '$date',
                        'revision_no' => ['$max' => '$revision_no'],
                    ]
                ],
                [
                    '$sort' => ['_id' => 1]
                ]
            ];

            $result = $collection->aggregate($query);
            foreach ($result as $row) {
                $max_revision = $row->revision_no;
            }
            return $max_revision;
        } catch (\Exception $e) {
            throw new \Exception($e->getMessage());
        }
    }
```

### To Update 

```php
public static function updateNetScheduleLtaReportAsDownloaded($netschedulelta_revision_no)
    {
        try {
            $date = date("Y-m-d", strtotime("+330 minutes"));
            $obj = new MongoDbHelper();
            $database = env('MONGO_DB_DATABASE');
            $collection = 'user_notifications';
            $namespace = $database . '.' . $collection;
            list($manager, $writeConcern, $bulk) = $obj->createBulkConnection();
            $datetime = date('Y-m-d H:i:s', strtotime("+330 minutes"));
            $updated_at = $obj->setMongoDate($datetime);
            $option = ["multi" => true, "upsert" => true];
            $user = Auth::user()->name;
            $filter = [
                'user' => $user,
                'netschedulelta_revision_no' => intval($netschedulelta_revision_no)
            ];
            $update = [
                '$set' => [
                    'inserted_at' => $updated_at,
                    'netschedulelta_download' => 1,
                ],
            ];
            $bulk->update($filter, $update, $option);
            $manager->executeBulkWrite($namespace, $bulk, $writeConcern);

        } catch (\Exception $e) {
            $msg = $e->getMessage();
            $status = 'failure';
        }
    }
```

### To Sort and Limit

```php
public static function selectData()
    {
        try {
            $response = [];
            $mongo = new MongoDbHelper();
            $conn = $mongo->createConnection();
            $db = $conn->selectDatabase(env('MONGO_DB_DATABASE'));
            $collection = $db->selectCollection("dayahead_accuracy_live");

            $query = [
                [
                    '$sort' => ['date' => -1]
                ],
                [
                    '$limit' => 7
                ],
            ];

            $result = $collection->aggregate($query);
            foreach ($result as $row) {
                $datetime = $mongo->getOnlyDateFromTimestamp($row->date);
                $response[] = [$datetime, round($row->ae, 2), round($row->avg_load, 2)];
            }
            return $response;
        } catch (\Exception $e) {
            $msg = $e->getMessage();
            $status = 'failure';
        }
```


```php
 $query = [
            [
                '$project' => [
                    'date' => 1,
                    'data' => ['$objectToArray' => '$data']
                ]
            ],
            [
                '$unwind' => '$data'
            ],
            [
                '$match' => [
                    'date' => [
                        '$gte' => $from_date,
                        '$lte' => $to_date
                    ]
                ]
            ],
            [
                '$sort' => ['date' => 1, 'data.v.created_at' => -1]
            ],
            ['$limit' => 30],
        ];
        $result = $collection->aggregate($query);
```

```php
$query = [
            [
                '$project' => [
                    'date'        => 1,
                    'entity_name' => 1,
                    'data'        => ['$objectToArray' => '$data']
                ]
            ],
            [
                '$unwind' => '$data'
            ],
            [
                '$match' => [
                    'date' => [
                        '$gte' => $from_date,
                        '$lte' => $to_date
                    ],
                    'entity_name' => 'Total'
                ]
            ],
            [
                '$sort' => ['date' => 1]
            ],
        ];
        $result = $collection->aggregate($query);
```

### Insert Data

```php
public static function insertData($user, $action)
    {
        try {
            date_default_timezone_set("UTC");
            $date = date("Y-m-d", strtotime("+330 minutes"));
            $obj = new MongoDbHelper();
            $database = env('MONGO_DB_DATABASE');
            $collection = 'user_logs';
            $namespace = $database . '.' . $collection;
            list($manager, $writeConcern, $bulk) = $obj->createBulkConnection();
            $datetime = date('Y-m-d H:i:s', strtotime("+330 minutes"));
            $updated_at = $obj->setMongoDate($datetime);
            $option = ["multi" => true, "upsert" => true];
            $filter = [
                'date' => $obj->setMongoDate($date),
            ];
            $update = [
                '$set' => [
                    'activity.' . $datetime => [
                        'inserted_at' => $updated_at,
                        'action' => $action,
                        'userid' => $user,
                    ],
                ]
            ];
            $bulk->update($filter, $update, $option);
            $manager->executeBulkWrite($namespace, $bulk, $writeConcern);

        } catch (\Exception $e) {
            $msg = $e->getMessage();
            $status = 'failure';
        }
    }
```

```php
    public function setData($data,$collection)
    {
        $response = [];
        $msg      = 'Data inserted Successfully';
        $status   = 'success';

        try
        {
            $obj = new MongoDbHelper();

            $database   = env('MONGO_DB_DATABASE');

            $namespace  = $database.'.'.$collection;

            list($manager,$writeConcern,$bulk) = $obj->createBulkConnection();

            $option = ["multi" => true, "upsert" => true];

            foreach ($data as $datum=>$values)
            {
                $filter =  [
                    'date'     => $obj->setMongoDate(date("Y-m-d",strtotime($values['date']))),
                    'revision' => intval($values['revision']),
                    'source'   => $values['source'],
                    'version'  => $values['version'],
                ];
                $update = [
                    '$set' => [
                        'data.'.$values['time'] => [
                            'datetime'  => $obj->setMongoDate(date('Y-m-d H:i:s',strtotime($values['date'].' '.$values['time']))),
                            'forecast'  => round($values['forecast'],2)
                        ],
                        'created_at'    => $obj->setMongoDate(date('Y-m-d H:i:s',strtotime("+330 minutes"))),
                    ]
                ];

                $bulk->update($filter,$update,$option);
            }
            $manager->executeBulkWrite($namespace, $bulk, $writeConcern);
        }
        catch (\Exception $e)
        {
            $msg = $e->getMessage();
            $status = 'failure';
        }
        $response['msg']    = $msg;
        $response['status'] = $status;
        return $response;
    }
```