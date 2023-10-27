# Advnaced fieltaring searching and more

## Basic Idea
- We want to be able to search for a specific field in a specific model
- We want to be able to search for a specific field in a specific model and sort the result
- We want to be able to search for a specific field in a specific model and sort the result and limit the result
- We want to be able to search for a specific field in a specific model and sort the result and limit the result and select specific fields
- We want to be able to search for a specific field in a specific model and sort the result and limit the result and select specific fields and paginate the result
- We want to be able to filter for a specific field in a specific model and sort the result and limit the result and select specific fields and paginate the result and filter the result

- Fot that purpose we will use a Class which name is APIFeatures

## APIFeatures Class
```ts
import { Document, FilterQuery, Query } from 'mongoose';
import { QueryStringInterface } from '../interfaces/common';

class APIFeatures<T extends Document> {
  constructor(
    private query: Query<T[], T>,
    private queryString: QueryStringInterface
  ) {}

  filter(): APIFeatures<T> {
    const queryObj = { ...this.queryString };
    const excludedFields = ['page', 'sort', 'limit', 'fields', 'searchText'];
    excludedFields.forEach(el => delete queryObj[el]);

    // Advanced filtering
    let queryStr = JSON.stringify(queryObj).replace(
      /\b(gte|gt|lte|lt)\b/g,
      match => `$${match}`
    );

    // Handle fields with multiple values
    Object.keys(queryObj).forEach(key => {
      const value = queryObj[key];
      if (Array.isArray(value)) {
        queryStr = queryStr.replace(
          `"${key}":${JSON.stringify(value)}`,
          `"${key}": { "$in": ${JSON.stringify(value)} }`
        );
      }
    });

    // Use where() instead of find()
    this.query = this.query.where(JSON.parse(queryStr));

    return this;
  }

  sort(): APIFeatures<T> {
    const sortBy = this.queryString.sort?.split(',').join(' ') ?? '-createdAt';
    this.query = this.query.sort(sortBy);
    return this;
  }

  limitFields(): APIFeatures<T> {
    const fields = this.queryString.fields?.split(',').join(' ') ?? '-__v';
    this.query = this.query.select(fields);
    return this;
  }

  paginate(): APIFeatures<T> {
    const page = +this.queryString.page || 1;
    const limit = +this.queryString.limit || 100;
    const skip = (page - 1) * limit;
    this.query = this.query.skip(skip).limit(limit);
    return this;
  }

  search(searchFields: (keyof T)[], searchText: string): APIFeatures<T> {
    if (searchText && searchFields.length > 0) {
      const orConditions: FilterQuery<T>[] = [];

      searchFields.forEach(field => {
        const condition = {
          [field]: { $regex: searchText, $options: 'i' },
        } as FilterQuery<T>;
        orConditions.push(condition);
      });

      // Combine search conditions with existing filter conditions using $and
      const existingConditions = this.query.getQuery();
      const combinedConditions: FilterQuery<T>[] = existingConditions.$and
        ? [...existingConditions.$and, { $or: orConditions }]
        : [{ $or: orConditions }];

      // Update the query with the combined conditions
      this.query = this.query.find({ $and: combinedConditions });
    }

    return this;
  }

  async exec(): Promise<T[]> {
    return this.query.exec();
  }

  getQuery(): FilterQuery<T> {
    return this.query.getQuery();
  }
  populate(path: string): APIFeatures<T> {
    this.query = this.query.populate(path);
    return this;
  }
}

export default APIFeatures;
```

## Interface
```ts
export interface QueryStringInterface {
  [key: string]: string;
}
```

## Folder Structure
```bash
├── src
│   ├── helpers
│   │   ├── apiFeatures.ts
│   ├── interfaces
│   │   ├── common.ts
```

## Making constants interface for the searching and filtering and more
```ts
//Testing CowSearchableFields (cow.constants.ts)
export const cowSearchableFieldsTest: (keyof ICow)[] = [
  'name',
  'location',
  'breed',
];


//Testing CowFilterableFields(cow.constants.ts)
export const cowFilterableFieldsTest = [
  'price',
  'location',
  'breed',
  'page',
  'limit',
  'sort',
  'fields',
  'searchText',
];
```

## Interface
```ts
//Testing Interface(cow.interface.ts)
export type ICowFiltersTest = {
  price?: string;
  location?: Location;
  breed?: Breed;
  searchText?: string;
  page?: string;
  limit?: string;
};
```


## Example : Cow Model and Folder structure
```bash
├── src
│   ├── modules
│   │   ├── cow
│   │   │   ├── cow.model.ts
│   │   │   ├── cow.controller.ts
│   │   │   ├── cow.service.ts
│   │   │   ├── cow.interface.ts
│   │   │   ├── cow.route.ts
│   │   │   ├── cow.constants.ts
```


## Example : Now ready for use first we need to create a service
```ts
//Testing getAllCowsDoc
const getAllCowsDocTest = async (
  filters: ICowFiltersTest
): Promise<IGenericResponse<ICow[]>> => {
  const query = Cow.find();
  const { searchText, page, limit } = filters;

  // Convert page and limit to numbers
  const pageNumber = page ? parseInt(page) : 1;
  const limitNumber = limit ? parseInt(limit) : 10;

  let features = new APIFeatures(query, filters)
    .filter()
    .sort()
    .limitFields()
    .paginate()
    .search([...cowSearchableFieldsTest], searchText as string)
    .populate('seller');
  
  const result = await features.exec();
  const total = await Cow.countDocuments(query.getFilter());

  return {
    meta: {
      page: pageNumber,
      limit: limitNumber,
      total,
    },
    data: result,
  };
};
```


## Example : Then we need to create a controller
```ts
const getAllCowDocsTest = catchAsync(async (req: Request, res: Response) => {
  const filters = pick(req.query, cowFilterableFieldsTest);
  // Convert comma separated values to array for multiple filters
  if (filters.breed && typeof filters.breed === 'string') {
    filters.breed = filters.breed.split(',');
  }
  const result = await CowService.getAllCowsDocTest(filters);
  sendResponse<ICow[]>(res, {
    statusCode: httpStatus.OK,
    success: true,
    message: 'Cow fetched successfully!',
    meta: result.meta,
    data: result.data,
  });
});
```


## Example : Then we need to create a route
```ts
//Testing getAllCowDocsTest
router.get(
  '/test',
  CowController.getAllCowDocsTest
);
```


## Testing api
```bash
https://digital-cow-hut-backend-one-zeta.vercel.app/api/v1/cows/test?price[gte]=5000&page=1&limit=10&location=Rajshahi&breed=Nellore,Brahman&sort=-breed&fields=name,price,seller&searchText=na (GET --> Filtering by price, location, breed(Multiple Fields), sorting, field selection, pagination, search)
```


## Thank you for reading this article
