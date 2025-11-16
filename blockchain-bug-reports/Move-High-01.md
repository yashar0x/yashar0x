# Unrestricted blob size leads to light node DoS via memory exhaustion

## Brief/Intro
An attacker can send an oversized blob request to the light node, leading to memory exhaustion and a crash, rendering the node unresponsive.

## Vulnerability Details
The light node processes new transaction batches in the form of Blobs using the `batch_write` function:
```rust
	/// Batch write blobs.
	async fn batch_write(
		&self,
		request: tonic::Request<grpc::BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<grpc::BatchWriteResponse>, tonic::Status> {
		let blobs_for_submission = request.into_inner().blobs;
```

The issue here is that there are no restrictions on the size of the blobs received by this function. As a result, an attacker can send an excessively large blob request to the light node, leading to memory exhaustion. This occurs because the node processes every transaction within the blob in a `for` loop:
```rust
	/// Batch write blobs.
	async fn batch_write(
		&self,
		request: tonic::Request<grpc::BatchWriteRequest>,
	) -> std::result::Result<tonic::Response<grpc::BatchWriteResponse>, tonic::Status> {
		let blobs_for_submission = request.into_inner().blobs;

		// make transactions from the blobs
		let mut transactions = Vec::new();
		for blob in blobs_for_submission {
			let transaction: Transaction = serde_json::from_slice(&blob.data)
				.map_err(|e| tonic::Status::internal(e.to_string()))?;

			match &self.prevalidator {
				Some(prevalidator) => {
					// match the prevalidated status, if validation error discard if internal error raise internal error
					match prevalidator.prevalidate(transaction).await {
						Ok(prevalidated) => {
							transactions.push(prevalidated.into_inner());
						}
						Err(e) => {
							match e {
								movement_da_light_node_prevalidator::Error::Validation(_) => {
									// discard the transaction
									info!(
										"discarding transaction due to prevalidation error {:?}",
										e
									);
								}
								movement_da_light_node_prevalidator::Error::Internal(e) => {
									return Err(tonic::Status::internal(e.to_string()));
								}
							}
						}
					}
				}
				None => transactions.push(transaction),
			}
		}
```

## Impact Details
An attacker can exploit this vulnerability by sending an extremely large blob, causing the node to run out of memory and crash. As a result, the affected node will become unresponsive and unable to process new transactions.

## References
None

## Proof of Concept
Detailed explanations inside **Vulnerability Details**.
