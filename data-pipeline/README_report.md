## Persistent Storage

### Object Store in CHI@TACC

`object-persist-project47`
`~7GB used`
[see on chi@tacc](https://chi.tacc.chameleoncloud.org/project/containers/container/object-persist-project47)

Contains:

1. **triplets_data** - Contains the PyTorch tensor file containing training data
2. **val_eval_batches** - For evaluation
3. **positives_splits**
4. **user_pairing_model**
5. **other_utils**

### Block Store in KVM@TACC

`block-persist-project47`
`Size 50GB allocated`
[see on kvm@tacc](https://kvm.tacc.chameleoncloud.org/project/volumes/42b1865a-6ff9-44ed-9f8d-8ca639f4b97c/)

Contains data for:

1. Postgres(used for storing feedback data, Metabase dashboard metadata, Grafana and Prometheus related data)
2. Grafana
3. Prometheus
4. MinIO

**Scripts for persistent storage setup**

- [Provision object store on Chameleon](./provision_object_storage.sh)
- [Provision block store on Chameleon](./provision_block_storage.sh)
- [Setup object store on the instance](./setup_object_store.sh)
- [Setup block store on the instance](./setup_block_store.sh)

## Offline data

> Needs some input from Agustin

## Data Pipeline

## Data Dashboard

We use a Metabase dashboard to monitor various metrics related to both the data and its quality.

[Docker compose for Metabase](./../docker/docker-compose-base.yaml)

For the training data, we display key metrics such as the number of distinct users and the average number of user-blocks per user.
A `user-block` is defined as a triplet: `(user_id, playlist_id_positive, playlist_id_negative_list)`. Each user-block groups together entries by user_id and their preferred `playlist_id_positive`, along with a list of `playlist_id_negative` values. These negative playlist IDs represent alternatives that the user preferred less than the positive one.
To ensure the integrity and consistency of our training data, we validate that each `user-block` contains only one `playlist_id_positive` per `(user_id, playlist_id_positive)` combination.

![training-widgets](./images/training.png)

For feedback data, we track metrics such as:

- Daily and weekly user counts providing feedback
- Average feedback score per day, to identify potential recommendation bias
- Invalid user-blocks, similar to training data checks, to ensure data integrity
- Feedback volume over time, which helps determine when enough new data has accumulated to trigger model retraining
- Cumulative feedback growth, providing insight into overall dataset expansion
- Number of distinct feedback users, which helps us monitor user diversity and avoid over-representation by "power users"

![feedback-widgets](./images/feedback.png)

## Online Data

[Airflow script for user simulation](./../model_monitoring/dags/pipeline_2_inference.py)

User interaction with the recommendation system is simulated using an **Airflow script**. This script utilizes the **test data split** stored in an object store and runs as a scheduled job every **10 minutes**. In each run, it randomly selects a subset of users and invokes the FastAPI `/recommend` endpoint over a 10-minute window to fetch playlist recommendations. Each API call returns **5 playlist recommendations** for a given user.

To **close the feedback loop**, we collect user responses through a hosted UI. Currently, the interface allows the user to **like one playlist out of the five recommended**. For simulation purposes, this interaction is automated— the Airflow script also makes calls to the FastAPI `/feedback` endpoint, mimicking user input. The simulated feedback is then stored in a **PostgreSQL database**, ensuring we can track and analyze user preferences over time.

The request and response format as given below:

**/recommend API**

Request

```
curl -X 'POST' \
  'http://<url>/api/playlist/recommend/' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "user_ids": [
    22434
  ]
}'
```

Response

```
[
  {
    "user_id": 22434,
    "playlists": [
      14546, 28783, 209745, 29747, 32649
    ]
  }
]
```

**/feedback API**

Request

```
curl -X 'POST' \
  'http://<url>/feedback' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "user_id": 22434,
  "like_playlist": 28783,
  "other_playlists": [
    14546, 209745, 29747, 32649
  ]
}'
```

Response

```
{
  "msg": "Feedback Sent"
}
```

## Data for Retraining

[Airflow script to extract feedback data for retraining](./../model_monitoring/dags/pipeline_extract_prod_data.py)

The **user feedback data stored in PostgreSQL** is extracted on a weekly basis using a separate Airflow script. This script processes the data by splitting it into **training, validation, and test sets**, and converts it into the appropriate format required for retraining the model.

To store the processed data, the script uses the rclone tool (invoked via Python subprocesses) to upload the new datasets to the **object store**. This newly collected data is then appended to the existing training dataset.

In recommendation systems, it's common to adopt a **sliding window approach** for training data. As we accumulate more feedback, older data is gradually phased out to keep the dataset manageable in size and relevant. This is especially important in our context—since user preferences tend to evolve, recent behavior (e.g., from the past month) is generally more indicative of current interests than data that is several months or a year old.
