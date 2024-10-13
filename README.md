# DigitalTolk Code Refactoring

## Booking Controller

### index()

 In booking controller I refactored index method. Each logical block of the original code is now in its own method, making it easier to read and maintain. The purpose of each method is explicit, enhancing readability. Returning early from methods that may not need to continue helps clarify the flow. Method names clearly describe their purpose.
 
 **Orginal Code**

```php
public function index(Request $request)
{
    if($user_id = $request->get('user_id')) {
        $response = $this->repository->getUsersJobs($user_id);
    } elseif($request->__authenticatedUser->user_type == env('ADMIN_ROLE_ID') || $request->__authenticatedUser->user_type == env('SUPERADMIN_ROLE_ID'))
    {
        $response = $this->repository->getAll($request);
    }

    return response($response);
}
```

**Refactor Code**

```php
public function index(Request $request)
{
    if ($user_id == $request->get('user_id')) {
        return $this->handleUserJobs($userId);
    }

    if ($this->isAdmin($request->__authenticatedUser)) {
        return $this->handleAdminJobs($request);
    }

    return response()->json(['message' => 'Unauthorized'], 403);
}
```

```php
protected function handleUserJobs($userId)
{
    $response = $this->repository->getUsersJobs($userId);
    return response($response);
}
```

```php
protected function handleAdminJobs(Request $request)
{
    $response = $this->repository->getAll($request);
    return response($response);
}
```

```php
protected function isAdmin($user)
{
    return in_array($user->user_type, [
        env('ADMIN_ROLE_ID'), 
        env('SUPERADMIN_ROLE_ID')
    ]);
}
```
## Booking Repository

### getUsersJobs()

In BookingRepository I refactored getUsersJobs method. The logic to get jobs by user type has been organized into the getJobsByUserType method. Similarly a method getUserType is created to get user type. The categorizeJobs method returns an array of emergency and normal jobs. The logic to check normal jobs is defined in addUserCheckToNormalJobs method. Each logical block of the original code is now in its own method

**Original Code**

```php
public function getUsersJobs($user_id)
{
    $cuser = User::find($user_id);
    $usertype = '';
    $emergencyJobs = array();
    $noramlJobs = array();
    if ($cuser && $cuser->is('customer')) {
        $jobs = $cuser->jobs()->with('user.userMeta', 'user.average', 'translatorJobRel.user.average', 'language', 'feedback')->whereIn('status', ['pending', 'assigned', 'started'])->orderBy('due', 'asc')->get();
        $usertype = 'customer';
    } elseif ($cuser && $cuser->is('translator')) {
        $jobs = Job::getTranslatorJobs($cuser->id, 'new');
        $jobs = $jobs->pluck('jobs')->all();
        $usertype = 'translator';
    }
    if ($jobs) {
        foreach ($jobs as $jobitem) {
            if ($jobitem->immediate == 'yes') {
                $emergencyJobs[] = $jobitem;
            } else {
                $noramlJobs[] = $jobitem;
            }
        }
        $noramlJobs = collect($noramlJobs)->each(function ($item, $key) use ($user_id) {
            $item['usercheck'] = Job::checkParticularJob($user_id, $item);
        })->sortBy('due')->all();
    }

    return ['emergencyJobs' => $emergencyJobs, 'noramlJobs' => $noramlJobs, 'cuser' => $cuser, 'usertype' => $usertype];
}
```
**Refactor Code**

```php
public function getUsersJobs($user_id)
{
    $cuser = User::find($user_id);
    $usertype = '';
    $emergencyJobs = [];
    $normalJobs = [];

    if(isset($cuser)) {
        $jobs = $this->getJobsByUserType($cuser);
        $usertype = $this->getUserType($cuser);

        if(isset($jobs)) {
            list($emergencyJobs, $normalJobs) = $this->categorizeJobs($jobs);
            $normalJobs = $this->addUserCheckToNormalJobs($normalJobs, $user_id);
        }
    }
    
    return [
        'emergencyJobs' => $emergencyJobs,
        'normalJobs' => $normalJobs,
        'cuser' => $cuser,
        'usertype' => $usertype,
    ];
}
```
```php
protected function getJobsByUserType($user)
{
    if ($user->is('customer')) {
        return $user->jobs()
            ->with('user.userMeta', 'user.average', 'translatorJobRel.user.average', 'language', 'feedback')
            ->whereIn('status', ['pending', 'assigned', 'started'])
            ->orderBy('due', 'asc')
            ->get();
    } elseif ($user->is('translator')) {
        $jobs = Job::getTranslatorJobs($user->id, 'new');
        return $jobs->pluck('jobs')->all();
    }

    return [];
}
```
```php
protected function getUserType($user)
{
    if ($user->is('customer')) {
        return 'customer';
    } elseif ($user->is('translator')) {
        return 'translator';
    }
    
    return '';
}
```
```php
protected function categorizeJobs($jobs)
{
    $emergencyJobs = [];
    $normalJobs = [];

    foreach ($jobs as $jobItem) {
        if ($jobItem->immediate == 'yes') {
            $emergencyJobs[] = $jobItem;
        } else {
            $normalJobs[] = $jobItem;
        }
    }

    return [$emergencyJobs, $normalJobs];
}
```
```php
protected function addUserCheckToNormalJobs($normalJobs, $user_id)
{
    return collect($normalJobs)->each(function ($item) use ($user_id) {
        $item['usercheck'] = Job::checkParticularJob($user_id, $item);
    })->sortBy('due')->all();
}
```

### store()

In BookingRepository I also refactored store method. Each logical block of code has been moved to its own method. The validation logic is encapsulated in the validateDate method, which checks required fields and returns a response if any are missing. The logic to prepare job data has been organized into the prepareJobData and related methods. Additional methods handle specific logic like determining gender, certification and job type, making code more modular.

**Original Code**

```php
public function store($user, $data)
{

    $immediatetime = 5;
    $consumer_type = $user->userMeta->consumer_type;
    if ($user->user_type == env('CUSTOMER_ROLE_ID')) {
        $cuser = $user;

        if (!isset($data['from_language_id'])) {
            $response['status'] = 'fail';
            $response['message'] = "Du måste fylla in alla fält";
            $response['field_name'] = "from_language_id";
            return $response;
        }
        if ($data['immediate'] == 'no') {
            if (isset($data['due_date']) && $data['due_date'] == '') {
                $response['status'] = 'fail';
                $response['message'] = "Du måste fylla in alla fält";
                $response['field_name'] = "due_date";
                return $response;
            }
            if (isset($data['due_time']) && $data['due_time'] == '') {
                $response['status'] = 'fail';
                $response['message'] = "Du måste fylla in alla fält";
                $response['field_name'] = "due_time";
                return $response;
            }
            if (!isset($data['customer_phone_type']) && !isset($data['customer_physical_type'])) {
                $response['status'] = 'fail';
                $response['message'] = "Du måste göra ett val här";
                $response['field_name'] = "customer_phone_type";
                return $response;
            }
            if (isset($data['duration']) && $data['duration'] == '') {
                $response['status'] = 'fail';
                $response['message'] = "Du måste fylla in alla fält";
                $response['field_name'] = "duration";
                return $response;
            }
        } else {
            if (isset($data['duration']) && $data['duration'] == '') {
                $response['status'] = 'fail';
                $response['message'] = "Du måste fylla in alla fält";
                $response['field_name'] = "duration";
                return $response;
            }
        }
        if (isset($data['customer_phone_type'])) {
            $data['customer_phone_type'] = 'yes';
        } else {
            $data['customer_phone_type'] = 'no';
        }

        if (isset($data['customer_physical_type'])) {
            $data['customer_physical_type'] = 'yes';
            $response['customer_physical_type'] = 'yes';
        } else {
            $data['customer_physical_type'] = 'no';
            $response['customer_physical_type'] = 'no';
        }

        if ($data['immediate'] == 'yes') {
            $due_carbon = Carbon::now()->addMinute($immediatetime);
            $data['due'] = $due_carbon->format('Y-m-d H:i:s');
            $data['immediate'] = 'yes';
            $data['customer_phone_type'] = 'yes';
            $response['type'] = 'immediate';

        } else {
            $due = $data['due_date'] . " " . $data['due_time'];
            $response['type'] = 'regular';
            $due_carbon = Carbon::createFromFormat('m/d/Y H:i', $due);
            $data['due'] = $due_carbon->format('Y-m-d H:i:s');
            if ($due_carbon->isPast()) {
                $response['status'] = 'fail';
                $response['message'] = "Can't create booking in past";
                return $response;
            }
        }
        if (in_array('male', $data['job_for'])) {
            $data['gender'] = 'male';
        } else if (in_array('female', $data['job_for'])) {
            $data['gender'] = 'female';
        }
        if (in_array('normal', $data['job_for'])) {
            $data['certified'] = 'normal';
        }
        else if (in_array('certified', $data['job_for'])) {
            $data['certified'] = 'yes';
        } else if (in_array('certified_in_law', $data['job_for'])) {
            $data['certified'] = 'law';
        } else if (in_array('certified_in_helth', $data['job_for'])) {
            $data['certified'] = 'health';
        }
        if (in_array('normal', $data['job_for']) && in_array('certified', $data['job_for'])) {
            $data['certified'] = 'both';
        }
        else if(in_array('normal', $data['job_for']) && in_array('certified_in_law', $data['job_for']))
        {
            $data['certified'] = 'n_law';
        }
        else if(in_array('normal', $data['job_for']) && in_array('certified_in_helth', $data['job_for']))
        {
            $data['certified'] = 'n_health';
        }
        if ($consumer_type == 'rwsconsumer')
            $data['job_type'] = 'rws';
        else if ($consumer_type == 'ngo')
            $data['job_type'] = 'unpaid';
        else if ($consumer_type == 'paid')
            $data['job_type'] = 'paid';
        $data['b_created_at'] = date('Y-m-d H:i:s');
        if (isset($due))
            $data['will_expire_at'] = TeHelper::willExpireAt($due, $data['b_created_at']);
        $data['by_admin'] = isset($data['by_admin']) ? $data['by_admin'] : 'no';

        $job = $cuser->jobs()->create($data);

        $response['status'] = 'success';
        $response['id'] = $job->id;
        $data['job_for'] = array();
        if ($job->gender != null) {
            if ($job->gender == 'male') {
                $data['job_for'][] = 'Man';
            } else if ($job->gender == 'female') {
                $data['job_for'][] = 'Kvinna';
            }
        }
        if ($job->certified != null) {
            if ($job->certified == 'both') {
                $data['job_for'][] = 'normal';
                $data['job_for'][] = 'certified';
            } else if ($job->certified == 'yes') {
                $data['job_for'][] = 'certified';
            } else {
                $data['job_for'][] = $job->certified;
            }
        }

        $data['customer_town'] = $cuser->userMeta->city;
        $data['customer_type'] = $cuser->userMeta->customer_type;

        //Event::fire(new JobWasCreated($job, $data, '*'));

//            $this->sendNotificationToSuitableTranslators($job->id, $data, '*');// send Push for New job posting
    } else {
        $response['status'] = 'fail';
        $response['message'] = "Translator can not create booking";
    }

    return $response;

}
```

**Refactor Code**

```php
public function store($user, $data)
{
    $consumerType = $user->userMeta->consumer_type;

    if ($user->user_type !== env('CUSTOMER_ROLE_ID')) {
        return $this->unauthorizedResponse();
    }

    $validationResponse = $this->validateData($data);
    if ($validationResponse) {
        return $validationResponse;
    }

    $response = $this->prepareJobData($data, $user, $consumerType);
    $job = $user->jobs()->create($response);

    return $this->successResponse($job);
}
```
```php
protected function validateData($data)
{
    $requiredFields = [
        'from_language_id' => "Du måste fylla in alla fält",
        'due_date' => "Du måste fylla in alla fält",
        'due_time' => "Du måste fylla in alla fält",
        'customer_phone_type' => "Du måste göra ett val här",
        'duration' => "Du måste fylla in alla fält",
    ];

    foreach ($requiredFields as $field => $message) {
        if (!isset($data[$field]) || $data[$field] === '') {
            return [
                'status' => 'fail',
                'message' => $message,
                'field_name' => $field,
            ];
        }
    }

    return null;
}
```
```php
protected function prepareJobData($data, $user, $consumerType)
{
    $data['customer_phone_type'] = isset($data['customer_phone_type']) ? 'yes' : 'no';
    $data['customer_physical_type'] = isset($data['customer_physical_type']) ? 'yes' : 'no';

    $this->setDueDateAndType($data);

    $data['gender'] = $this->determineGender($data['job_for']);
    $data['certified'] = $this->determineCertification($data['job_for']);
    $data['job_type'] = $this->determineJobType($consumerType);
    $data['b_created_at'] = now();
    
    if (isset($data['due'])) {
        $data['will_expire_at'] = TeHelper::willExpireAt($data['due'], $data['b_created_at']);
    }

    $data['by_admin'] = $data['by_admin'] ?? 'no';
    return $data;
}
```
```php
protected function setDueDateAndType(&$data)
{
    if ($data['immediate'] === 'yes') {
        $data['due'] = Carbon::now()->addMinutes(5)->format('Y-m-d H:i:s');
        $data['immediate'] = 'yes';
    } else {
        $due = $data['due_date'] . " " . $data['due_time'];
        $dueCarbon = Carbon::createFromFormat('m/d/Y H:i', $due);
        
        if ($dueCarbon->isPast()) {
            return [
                'status' => 'fail',
                'message' => "Can't create booking in past",
            ];
        }
        
        $data['due'] = $dueCarbon->format('Y-m-d H:i:s');
        $data['immediate'] = 'no';
    }
}
```
```php
protected function determineGender($jobFor)
{
    if (in_array('male', $jobFor)) return 'male';
    if (in_array('female', $jobFor)) return 'female';
    return null;
}
```
```php
protected function determineCertification($jobFor)
{
    if (in_array('certified', $jobFor)) return 'yes';
    if (in_array('certified_in_law', $jobFor)) return 'law';
    if (in_array('certified_in_health', $jobFor)) return 'health';
    return 'normal';
}
```
```php
protected function determineJobType($consumerType)
{
    switch ($consumerType) {
        case 'rwsconsumer':
            return 'rws';
        case 'ngo':
            return 'unpaid';
        case 'paid':
            return 'paid';
        default:
            return null;
    }
}
```
```php
protected function successResponse($job)
{
    return [
        'status' => 'success',
        'id' => $job->id,
        'job_for' => $this->formatJobForResponse($job),
        'customer_town' => $job->userMeta->city,
        'customer_type' => $job->userMeta->customer_type,
    ];
}
```
```php
protected function formatJobForResponse($job)
{
    $jobFor = [];
    if ($job->gender) {
        $jobFor[] = $job->gender === 'male' ? 'Man' : 'Kvinna';
    }
    if ($job->certified) {
        $jobFor[] = $job->certified === 'both' ? 'normal' : $job->certified;
    }
    return $jobFor;
}
```
```php
protected function unauthorizedResponse()
{
    return [
        'status' => 'fail',
        'message' => "Translator can not create booking",
    ];
}
```