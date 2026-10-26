# The tenant is a customer account, not a store

Open Food Facts records a `stores_tags` list for each product. Thus one product sells at
several stores, the spellings in that one field are inconsistent, and many products name
no store at all. A store cannot be a foreign key on `products`. We model the tenant as one
customer account of this service, which is the data team of a retailer. Each tenant owns
its own product rows, and the same barcode can exist in two tenants with different images
attached.

## Consequences

The unique index on `products` is `(tenant_id, barcode)`, not `barcode`. Every read is
tenant-scoped, including `GET /products/{barcode}/images`. A request names its tenant with
a hashed `X-Tenant-Key` header. The seed script writes one product row per store in
`stores_tags`, so one Open Food Facts record can become several rows.
