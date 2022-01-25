# ACED-IDP Minio based file service

## Proof of concept

> Verify presigned urls can be used for upload and download

## Setup

* Setup dependencies

```
python3 -m venv venv
source venv/bin/activate
```

* Create a work directory and sample data

```
mkdir minio
cd minio
mkdir -p data/test-bucket
echo 'foo' > data/test-bucket/test.txt
```

* Launch the docker container

```
export MINIO_ROOT_USER=...
export MINIO_ROOT_PASSWORD=...

docker run \
 -p 9000:9000 \
 -p 9001:9001  \
 -v $(pwd)/data:/data \
 --name minio \
 --env MINIO_ROOT_USER --env MINIO_ROOT_PASSWORD \
 minio/minio server /data --console-address ":9001"
```

You should see:

```
API: http://172.17.0.2:9000  http://127.0.0.1:9000

Console: http://172.17.0.2:9001 http://127.0.0.1:9001

Documentation: https://docs.min.io
```

## Verify docker server

  * Create a python virtual environment
    ```
    python3 -m venv venv
    pip install -r requirements.txt
    ```
  
  * Configure the minio command line tool  `myminio`
    ```
    mc config host a myminio http://localhost:9000 ${MINIO_ROOT_USER} ${MINIO_ROOT_PASSWORD}
    >>> Added `myminio` successfully.
    ```
    
  * List the buckets & contents
    ```
    mc ls myminio
    >>> [2022-01-24 08:31:36 PST]     0B test-bucket/
    mc ls myminio/test-bucket/
    >>> [2022-01-24 08:33:45 PST]     4B test.txt
    ```

  * List via AWS cli

    * First install the amazon cli https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

    * Create a profile
    ```
    cat << EOF >>  ~/.aws/credentials

    [myminio]
    aws_access_key_id=$MINIO_ROOT_USER
    aws_secret_access_key=$MINIO_ROOT_PASSWORD

    EOF

    cat << EOF >>  ~/.aws/config

    [myminio]
    endpoint=http://localhost:9000
    signature_version=s3v4
    region=us-east-1

    EOF

    ```

    * List bucket contents
    ```
    aws --profile myminio --endpoint-url http://localhost:9000  s3 ls
    >>> 2022-01-24 08:31:36 test-bucket
    aws --profile myminio --endpoint-url http://localhost:9000  s3 ls test-bucket
    >>> 2022-01-24 08:33:45          4 test.txt
    ```

## Request pre-signed url to read file

    * We use the aws `presign` utility to get the presigned url
    ```
    aws --profile myminio --endpoint-url http://localhost:9000  s3 presign test-bucket/test.txt
    >>> http://localhost:9000/test-bucket/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20220124%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220124T165249Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=f66429baa861a71e20aced98902e4fa1773c1efdc720d7b640147d64fa6a6acf
    ```

    * Use wget or curl to read file

    ```
     wget 'http://localhost:9000/test-bucket/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20220124%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220124T165249Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=f66429baa861a71e20aced98902e4fa1773c1efdc720d7b640147d64fa6a6acf' -O /tmp/test.txt
    --2022-01-24 08:53:28--  http://localhost:9000/test-bucket/test.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20220124%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220124T165249Z&X-Amz-Expires=3600&X-Amz-SignedHeaders=host&X-Amz-Signature=f66429baa861a71e20aced98902e4fa1773c1efdc720d7b640147d64fa6a6acf
    Resolving localhost (localhost)... ::1, 127.0.0.1
    Connecting to localhost (localhost)|::1|:9000... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 4 [text/plain]
    Saving to: ‘/tmp/test.txt’

    /tmp/test.txt                                100%[============================================================================================>]       4  --.-KB/s    in 0s

    2022-01-24 08:53:28 (488 KB/s) - ‘/tmp/test.txt’ saved [4/4]
    ```

## Request pre-signed url to write a file

    * We use our `minio_presigned_put` utility to get the presigned url
    ```
    minio_presigned_put  --bucket_name test-bucket --object_name test2.txt
    http://localhost:9000/test-bucket/test2.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20220124%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220124T172148Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=6acf04ac46f433b664ff6c0dc4a1f5152515b53ee866fbf1f5ea87b37b95a1f7
    ```

    * Use curl to upload file
    ```
    cat << EOF > /tmp/test2.txt
    my test upload
    EOF    

    curl -v --upload-file /tmp/test2.txt 'http://localhost:9000/test-bucket/test2.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=minioadmin%2F20220124%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20220124T172148Z&X-Amz-Expires=604800&X-Amz-SignedHeaders=host&X-Amz-Signature=6acf04ac46f433b664ff6c0dc4a1f5152515b53ee866fbf1f5ea87b37b95a1f7'

    >>> HTTP/1.1 200 OK
    ```

    * Verify file exits

    ```
    aws --profile myminio --endpoint-url http://localhost:9000  s3 ls test-bucket
    2022-01-24 08:33:45          4 test.txt
    2022-01-24 09:25:26         15 test2.txt
    ```


## Next steps

    * multi part upload
        * https://vsoch.github.io/2020/s3-minio-multipart-presigned-upload/
        * https://dev.to/traindex/multipart-upload-for-large-files-using-pre-signed-urls-aws-4hg4
