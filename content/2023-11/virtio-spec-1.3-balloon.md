+++
title = "Virtio Spec v1.3 - Balloon"
date = "2023-11-22"
[extra]
add_toc = true

+++

*[source](https://docs.oasis-open.org/virtio/virtio/v1.3/csd01/virtio-v1.3-csd01.html)*

##  Traditional Memory Balloon Device

This is the traditional balloon device. The device number 13 is reserved for a new memory balloon interface, with different semantics, which is expected in a future version of the standard.

The traditional virtio memory balloon device is a primitive device for managing guest memory: the device asks for a certain amount of memory, and the driver supplies it (or withdraws it, if the device has more than it asks for). This allows the guest to adapt to changes in allowance of underlying physical memory. If the feature is negotiated, the device can also be used to communicate guest memory statistics to the host.

## 1 Device ID

5

## 2 Virtqueues

| Index | Queue          | Condition                                               |
| ----- | -------------- | ------------------------------------------------------- |
| 0     | `inflateq`     |                                                         |
| 1     | `deflateq`     |                                                         |
| 2     | `statsq`       | only exists if `VIRTIO_BALLOON_F_STATS_VQ` is set       |
| 3     | `free_page_vq` | only exists if `VIRTIO_BALLOON_F_FREE_PAGE_HINT` is set |
| 4     | `reporting_vq` | only exists if `VIRTIO_BALLOON_F_PAGE_REPORTING` is set |

## 3 Feature bits

| Bit  | Name                              | Meaning                                                      |
| ---- | --------------------------------- | ------------------------------------------------------------ |
| 0    | `VIRTIO_BALLOON_F_MUST_TELL_HOST` | Host has to be told before pages from the balloon are used.  |
| 1    | `VIRTIO_BALLOON_F_STATS_VQ`       | A virtqueue for reporting guest memory statistics is present. |
| 2    | `VIRTIO_BALLOON_F_DEFLATE_ON_OOM` | Deflate balloon on guest out of memory condition.            |
| 3    | `VIRTIO_BALLOON_F_FREE_PAGE_HINT` | The device has support for free page hinting. A virtqueue for providing hints as to what memory is currently free is present. Configuration field *free_page_hint_cmd_id* is valid. |
| 4    | `VIRTIO_BALLOON_F_PAGE_POISON`    | A hint to the device, that the driver will immediately write *poison_val* to pages after deflating them. Configuration field *poison_val* is valid. |
| 5    | `VIRTIO_BALLOON_F_PAGE_REPORTING` | The device has support for free page reporting. A virtqueue for reporting free guest memory is present. |

### 3.1 Driver Requirements: Feature bits

The driver SHOULD accept the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature if offered by the device.

The driver SHOULD clear the `VIRTIO_BALLOON_F_PAGE_POISON` flag if it will not immediately write *poison_val* to deflated pages (e.g., to initialize them, or fill them with a poison value).

If the driver is expecting the pages to retain some initialized value, it MUST NOT accept `VIRTIO_BALLOON_F_PAGE_REPORTING` unless it also negotiates `VIRTIO_BALLOON_F_PAGE_POISON`.

### 3.2 Device Requirements: Feature bits

If the device offers the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature bit, and if the driver did not accept this feature bit, the device MAY signal failure by failing to set `FEATURES_OK` *device status* bit when the driver writes it.



#### 3.2.0.1 Legacy Interface: Feature bits

As the legacy interface does not have a way to gracefully report feature negotiation failure, when using the legacy interface, transitional devices MUST support guests which do not negotiate `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature, and SHOULD allow guest to use memory before notifying host if `VIRTIO_BALLOON_F_MUST_TELL_HOST` is not negotiated.

## 4 Device configuration layout

*num_pages* and *actual* are always available.

*free_page_hint_cmd_id* is available if `VIRTIO_BALLOON_F_FREE_PAGE_HINT` has been negotiated. The field is read-only by the driver. *poison_val* is available if `VIRTIO_BALLOON_F_PAGE_POISON` has been negotiated.

```c
struct virtio_balloon_config { 
    le32 num_pages; 
    le32 actual; 
    le32 free_page_hint_cmd_id; 
    le32 poison_val; 
};
```





#### 4.0.0.1 Legacy Interface: Device configuration layout

When using the legacy interface, transitional devices and drivers MUST format the fields in `struct virtio_balloon_config` according to the little-endian format. Note: This is unlike the usual convention that legacy device fields are guest endian.

## 5 Device Initialization

The device initialization process is outlined below:

1. The inflate and deflate virtqueues are identified.

2. If the `VIRTIO_BALLOON_F_STATS_VQ` feature bit is negotiated:
   1. Identify the stats virtqueue.

   2. Add one empty buffer to the stats virtqueue.

3. If the `VIRTIO_BALLOON_F_FREE_PAGE_HINT` feature bit is negotiated, identify the `free_page_vq`.

4. If the `VIRTIO_BALLOON_F_PAGE_POISON` feature bit is negotiated, update the *poison_val* configuration field.

5. If the `VIRTIO_BALLOON_F_PAGE_REPORTING` feature bit is negotiated, identify the `reporting_vq`.

6. `DRIVER_OK` is set: device operation begins.

7. If the `VIRTIO_BALLOON_F_STATS_VQ` feature bit is negotiated, then notify the device about the stats virtqueue buffer.

8. If the `VIRTIO_BALLOON_F_PAGE_REPORTING` feature bit is negotiated, then begin reporting free pages to the device.



## 6 Device Operation

The device is driven either by the receipt of a configuration change notification, or by changing guest memory needs, such as performing memory compaction or responding to out of memory conditions.

1. *num_pages* configuration field is examined. If this is greater than the *actual* number of pages, the balloon wants more memory from the guest. If it is less than *actual*, the balloon doesn’t need it all.

2. To supply memory to the balloon (aka. inflate): (a) The driver constructs an array of addresses of unused memory pages. These addresses are divided by 4096[16](https://docs.oasis-open.org/virtio/virtio/v1.3/csd01/virtio-v1.3-csd01.html#fn16x5) and the descriptor describing the resulting 32-bit array is added to the `inflateq`.

3. To remove memory from the balloon (aka. deflate):
   1. The driver constructs an array of addresses of memory pages it has previously given to the balloon, as described above. This descriptor is added to the deflateq.

   2. If the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is negotiated, the guest informs the device of pages before it uses them.

   3. Otherwise, the guest is allowed to re-use pages previously given to the balloon before the device has acknowledged their withdrawal[17](https://docs.oasis-open.org/virtio/virtio/v1.3/csd01/virtio-v1.3-csd01.html#fn17x5).

4. In either case, the device acknowledges inflate and deflate requests by using the descriptor.

5. Once the device has acknowledged the inflation or deflation, the driver updates *actual* to reflect the new number of pages in the balloon.



### 6.1 Driver Requirements: Device Operation

The driver SHOULD supply pages to the balloon when *num_pages* is greater than the actual number of pages in the balloon.

The driver MAY use pages from the balloon when *num_pages* is less than the actual number of pages in the balloon.

The driver MAY supply pages to the balloon when *num_pages* is greater than or equal to the actual number of pages in the balloon.

If `VIRTIO_BALLOON_F_DEFLATE_ON_OOM` has not been negotiated, the driver MUST NOT use pages from the balloon when *num_pages* is less than or equal to the actual number of pages in the balloon.

If `VIRTIO_BALLOON_F_DEFLATE_ON_OOM` has been negotiated, the driver MAY use pages from the balloon when *num_pages* is less than or equal to the actual number of pages in the balloon if this is required for system stability (e.g. if memory is required by applications running within the guest).

The driver MUST use the deflateq to inform the device of pages that it wants to use from the balloon.

If the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is negotiated, the driver MUST NOT use pages from the balloon until the device has acknowledged the deflate request.

Otherwise, if the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is not negotiated, the driver MAY begin to re-use pages previously given to the balloon before the device has acknowledged the deflate request.

In any case, the driver MUST NOT use pages from the balloon after adding the pages to the balloon, but before the device has acknowledged the inflate request.

The driver MUST NOT request deflation of pages in the balloon before the device has acknowledged the inflate request.

The driver MUST update *actual* after changing the number of pages in the balloon.

The driver MAY update *actual* once after multiple inflate and deflate operations.

### 6.2 Device Requirements: Device Operation

The device MAY modify the contents of a page in the balloon after detecting its physical number in an inflate request and before acknowledging the inflate request by using the inflateq descriptor.

If the `VIRTIO_BALLOON_F_MUST_TELL_HOST` feature is negotiated, the device MAY modify the contents of a page in the balloon after detecting its physical number in an inflate request and before detecting its physical number in a deflate request and acknowledging the deflate request.



### 6.2.1 Legacy Interface: Device Operation

When using the legacy interface, the driver SHOULD ignore the used length values. Note: Historically, some devices put the total descriptor length there, even though no data was actually written.

When using the legacy interface, the driver MUST write out all 4 bytes each time it updates the *actual* value in the configuration space, using a single atomic operation.

When using the legacy interface, the device SHOULD NOT use the *actual* value written by the driver in the configuration space, until the last, most-significant byte of the value has been written. Note: Historically, devices used the *actual* value, even though when using Virtio Over PCI Bus the device-specific configuration space was not guaranteed to be atomic. Using intermediate values during update by driver is best avoided, except for debugging.

Historically, drivers using Virtio Over PCI Bus wrote the *actual* value by using multiple single-byte writes in order, from the least-significant to the most-significant value.

### 6.3 Memory Statistics

The stats virtqueue is atypical because communication is driven by the device (not the driver). The channel becomes active at driver initialization time when the driver adds an empty buffer and notifies the device. A request for memory statistics proceeds as follows:

1. The device uses the buffer and sends a used buffer notification.

2. The driver pops the used buffer and discards it.

3. The driver collects memory statistics and writes them into a new buffer.

4. The driver adds the buffer to the virtqueue and notifies the device.

5. The device pops the buffer (retaining it to initiate a subsequent request) and consumes the statistics.

Within the buffer, statistics are an array of 10-byte entries. Each statistic consists of a 16 bit tag and a 64 bit value. All statistics are optional and the driver chooses which ones to supply. To guarantee backwards compatibility, devices omit unsupported statistics.

```c
struct virtio_balloon_stat { 
#define VIRTIO_BALLOON_S_SWAP_IN 0 
#define VIRTIO_BALLOON_S_SWAP_OUT 1 
#define VIRTIO_BALLOON_S_MAJFLT  2 
#define VIRTIO_BALLOON_S_MINFLT  3 
#define VIRTIO_BALLOON_S_MEMFREE 4 
#define VIRTIO_BALLOON_S_MEMTOT  5 
#define VIRTIO_BALLOON_S_AVAIL  6 
#define VIRTIO_BALLOON_S_CACHES  7 
#define VIRTIO_BALLOON_S_HTLB_PGALLOC 8 
#define VIRTIO_BALLOON_S_HTLB_PGFAIL 9 
    le16 tag; 
    le64 val; 
} __attribute__((packed));
```




### 6.3.1 Driver Requirements: Memory Statistics

Normative statements in this section apply if and only if the `VIRTIO_BALLOON_F_STATS_VQ` feature has been negotiated.

The driver MUST make at most one buffer available to the device in the `statsq`, at all times.

After initializing the device, the driver MUST make an output buffer available in the `statsq`.

Upon detecting that device has used a buffer in the `statsq`, the driver MUST make an output buffer available in the `statsq`.

Before making an output buffer available in the `statsq`, the driver MUST initialize it, including one `struct virtio_balloon_stat` entry for each statistic that it supports.

Driver MUST use an output buffer size which is a multiple of 6 bytes for all buffers submitted to the `statsq`.

Driver MAY supply `struct virtio_balloon_stat` entries in the output buffer submitted to the `statsq` in any order, without regard to *tag* values.

Driver MAY supply a subset of all statistics in the output buffer submitted to the `statsq`.

Driver MUST supply the same subset of statistics in all buffers submitted to the `statsq`.



### 6.3.2 Device Requirements: Memory Statistics

Normative statements in this section apply if and only if the `VIRTIO_BALLOON_F_STATS_VQ` feature has been negotiated.

Within an output buffer submitted to the `statsq`, the device MUST ignore entries with *tag* values that it does not recognize.

Within an output buffer submitted to the `statsq`, the device MUST accept `struct virtio_balloon_stat` entries in any order without regard to *tag* values.



### 6.3.3 Legacy Interface: Memory Statistics

When using the legacy interface, transitional devices and drivers MUST format the fields in struct virtio_balloon_stat according to the native endian of the guest rather than (necessarily when not using the legacy interface) little-endian.

When using the legacy interface, the device SHOULD ignore all values in the first buffer in the statsq supplied by the driver after device initialization. Note: Historically, drivers supplied an uninitialized buffer in the first buffer.

### 6.4 Memory Statistics Tags

| Value | Name                            | Meaning                                                      |
| ----- | ------------------------------- | ------------------------------------------------------------ |
| 0     | `VIRTIO_BALLOON_S_SWAP_IN`      | The amount of memory that has been swapped in (in bytes).    |
| 1     | `VIRTIO_BALLOON_S_SWAP_OUT`     | The amount of memory that has been swapped out to disk (in bytes). |
| 2     | `VIRTIO_BALLOON_S_MAJFLT`       | The number of major page faults that have occurred.          |
| 3     | `VIRTIO_BALLOON_S_MINFLT`       | The number of minor page faults that have occurred.          |
| 4     | `VIRTIO_BALLOON_S_MEMFREE`      | The amount of memory not being used for any purpose (in bytes). |
| 5     | `VIRTIO_BALLOON_S_MEMTOT`       | The total amount of memory available (in bytes).             |
| 6     | `VIRTIO_BALLOON_S_AVAIL`        | An estimate of how much memory is available (in bytes) for starting new applications, without pushing the system to swap. |
| 7     | `VIRTIO_BALLOON_S_CACHES`       | The amount of memory, in bytes, that can be quickly reclaimed without additional I/O. Typically these pages are used for caching files from disk. |
| 8     | `VIRTIO_BALLOON_S_HTLB_PGALLOC` | The number of successful hugetlb page allocations in the guest. |
| 9     | `VIRTIO_BALLOON_S_HTLB_PGFAIL`  | The number of failed hugetlb page allocations in the guest.  |

### 6.5 Free Page Hinting

Free page hinting is designed to be used during migration to determine what pages within the guest are currently unused so that they can be skipped over while migrating the guest. The device will indicate that it is ready to start performing hinting by setting the *free_page_hint_cmd_id* to one of the non-reserved values that can be used as a command ID. The following values are reserved:

- `VIRTIO_BALLOON_CMD_ID_STOP` (0)

  Any command ID previously supplied by the device is invalid. The driver should stop hinting free pages until a new command ID is supplied, but should not release any hinted pages for use by the guest.

- `VIRTIO_BALLOON_CMD_ID_DONE` (1)

  Any command ID previously supplied by the device is invalid. The driver should stop hinting free pages, and should release all hinted pages for use by the guest.

When a hint is provided by the driver it indicates that the data contained in the given page is no longer needed and can be discarded. If the driver writes to the page this overrides the hint and the data will be retained. The contents of any stale pages that have not been written to since the page was hinted may be lost, and if read the contents of such pages will be uninitialized memory.

A request for free page hinting proceeds as follows:

1. The driver examines the *free_page_hint_cmd_id* configuration field. If it contains a non-reserved value then free page hinting will begin.

2. To supply free page hints:
   1. The driver constructs an output buffer containing the new value from the *free_page_hint_cmd_id* configuration field and adds it to the free_page_vq.

   2. The driver maps a series of pages and adds them to the free_page_vq as individual scatter-gather input buffer entries.

   3. When the driver is no longer able to fetch additional pages to add to the free_page_vq, it will construct an output buffer containing the command ID `VIRTIO_BALLOON_CMD_ID_STOP`.

3. A round of hinting ends either when the driver is no longer able to supply more pages for hinting as described above, or when the device updates *free_page_hint_cmd_id* configuration field to contain either `VIRTIO_BALLOON_CMD_ID_STOP` or `VIRTIO_BALLOON_CMD_ID_DONE`.

4. The device may follow `VIRTIO_BALLOON_CMD_ID_STOP` with a new non-reserved value for the *free_page_hint_cmd_id* configuration field in which case it will resume supplying free page hints.

5. Otherwise, if the device provides `VIRTIO_BALLOON_CMD_ID_DONE` then hinting is complete and the driver may release all previously hinted pages for use by the guest.





### 6.5.1 Driver Requirements: Free Page Hinting

Normative statements in this section apply if the `VIRTIO_BALLOON_F_FREE_PAGE_HINT` feature has been negotiated.

The driver MUST use an output buffer size of 4 bytes for all output buffers submitted to the `free_page_vq`.

The driver MUST start hinting by providing an output buffer containing the current command ID for the given block of pages.

The driver MUST NOT provide more than one output buffer containing the current command ID.

The driver SHOULD supply pages to the `free_page_vq` as input buffers when *free_page_hint_cmd_id* specifies a value of 2 or greater.

The driver SHOULD stop supplying pages for hinting when *free_page_hint_cmd_id* specifies a value of `VIRTIO_BALLOON_CMD_ID_STOP` or `VIRTIO_BALLOON_CMD_ID_DONE`.

If the driver is unable to supply pages, it MUST complete hinting by adding an output buffer containing the command ID `VIRTIO_BALLOON_CMD_ID_STOP`.

The driver MAY release hinted pages for use by the guest including when the device has not yet used the descriptor containing the hinting request.

The driver MUST treat the content of all hinted pages as uninitialized memory.

The driver MUST initialize the contents of any previously hinted page released before *free_page_hint_cmd_id* specifies a value of `VIRTIO_BALLOON_CMD_ID_DONE`.

The driver SHOULD release all previously hinted pages once *free_page_hint_cmd_id* specifies a value of `VIRTIO_BALLOON_CMD_ID_DONE`.



### 6.5.2 Device Requirements: Free Page Hinting

Normative statements in this section apply if the `VIRTIO_BALLOON_F_FREE_PAGE_HINT` feature has been negotiated.

The device SHOULD set *free_page_hint_cmd_id* to `VIRTIO_BALLOON_CMD_ID_STOP` any time that it will not be able to make use of the hints provided by the driver.

The device MUST NOT reuse a command ID until it has received an output buffer containing `VIRTIO_BALLOON_CMD_ID_STOP` from the driver.

The device MUST ignore pages that are provided with a command ID that does not match the current value in *free_page_hint_cmd_id*.

If the content of a previously hinted page has not been modified by the guest since the device issued the *free_page_hint_cmd_id* associated with the hint, the device MAY modify the contents of the page.

The device MUST NOT modify the content of a previously hinted page after *free_page_hint_cmd_id* is set to `VIRTIO_BALLOON_CMD_ID_DONE`.

The device MUST report a value of `VIRTIO_BALLOON_CMD_ID_DONE` in *free_page_hint_cmd_id* when it no longer has need for the previously hinted pages.



### 6.5.3 Legacy Interface: Free Page Hinting

When using the legacy interface, transitional devices and drivers MUST format the command ID field in output buffers according to the native endian of the guest rather than (necessarily when not using the legacy interface) little-endian.

### 6.6 Page Poison

Page Poison provides a way to notify the host that the guest is initializing free pages with *poison_val*. When the feature is enabled, pages will be immediately written to by the driver after deflating, and pages reported by free page reporting will retain the value indicated by *poison_val*.

If the guest is not initializing freed pages, the driver should reject the `VIRTIO_BALLOON_F_PAGE_POISON` feature.

If `VIRTIO_BALLOON_F_PAGE_POISON` feature has been negotiated, the driver will place the initialization value into the *poison_val* configuration field data.



### 6.6.1 Driver Requirements: Page Poison

Normative statements in this section apply if the `VIRTIO_BALLOON_F_PAGE_POISON` feature has been negotiated.

The driver MUST initialize the deflated pages with *poison_val* when they are reused by the driver.

The driver MUST populate the *poison_val* configuration data before setting the `DRIVER_OK` bit.

The driver MUST NOT modify *poison_val* while the `DRIVER_OK` bit is set.



### 6.6.2 Device Requirements: Page Poison

Normative statements in this section apply if the `VIRTIO_BALLOON_F_PAGE_POISON` feature has been negotiated.

The device MAY use the content of *poison_val* as a hint to guest behavior.

### 6.7 Free Page Reporting

Free Page Reporting provides a mechanism similar to balloon inflation, however it does not provide a deflation queue. Reported free pages can be reused by the driver after the reporting request has been acknowledged without notifying the device.

The driver will begin reporting free pages. When exactly and which free pages are reported is up to the driver.

1. The driver determines it has enough pages available to begin reporting free pages.

2. The driver gathers free pages into a scatter-gather list and adds them to the `reporting_vq`.

3. The device acknowledges the reporting request by using the `reporting_vq` descriptor.

4. Once the device has acknowledged the report, the driver can reuse the reported free pages when needed (e.g., by putting them back to free page lists in the guest operating system).

5. The driver can then continue to gather and report free pages until it has determined it has reported a sufficient quantity of pages.





### 6.7.1 Driver Requirements: Free Page Reporting

Normative statements in this section apply if the `VIRTIO_BALLOON_F_PAGE_REPORTING` feature has been negotiated.

If the `VIRTIO_BALLOON_F_PAGE_POISON` feature has not been negotiated, then the driver MUST treat all reported pages as uninitialized memory.

If the `VIRTIO_BALLOON_F_PAGE_POISON` feature has been negotiated, the driver MUST initialize all free pages with *poison_val* before reporting them.

The driver MUST NOT use the reported pages until the device has acknowledged the reporting request.

The driver MAY report free pages any time after `DRIVER_OK` is set.

The driver SHOULD attempt to report large pages rather than smaller ones.

The driver SHOULD avoid reading/writing reported pages if not strictly necessary.



### 6.7.2 Device Requirements: Free Page Reporting

Normative statements in this section apply if the `VIRTIO_BALLOON_F_PAGE_REPORTING` feature has been negotiated.

If the `VIRTIO_BALLOON_F_PAGE_POISON` feature has not been negotiated, the device MAY modify the contents of any page supplied in a report request before acknowledging that request by using the `reporting_vq` descriptor.

If the `VIRTIO_BALLOON_F_PAGE_POISON` feature has been negotiated, the device MUST NOT modify the the content of a reported page to a value other than *poison_val*.