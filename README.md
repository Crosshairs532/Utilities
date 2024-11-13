> [!IMPORTANT]
> # Custom Functions
> ### Config
```javascript
import dotenv from 'dotenv';
import path from 'path';

dotenv.config({ path: path.join((process.cwd(), '.env')) });
export default {
  node_env: process.env.NODE_ENV,
  port: process.env.PORT,
  database_url: process.env.DB_URL as string,
  defaultPassword: process.env.defaultPassword,
  saltround: process.env.SALT_ROUND,
  jwt_secret: process.env.jwt_secret,
};

```
> ### Server
```javascript
import mongoose from 'mongoose';
import app from './app';
import config from './app/config';
import { Server } from 'http';
let server: Server;
async function main() {
  try {
    await mongoose.connect(config.database_url as string);
    server = app.listen(config.port, () => {
      console.log(`app is listening on port ${config.port}`);
    });
  } catch (err) {
    console.log(err);
  }
}
main();
// for asynchronous  behavior
process.on('unhandledRejection', () => {
  console.log(`UnhandledRejection is detected, shutting down server....`);
  if (server) {
    server.close(() => {
      process.exit(1);
    });
  }
  process.exit(1);
});
//  for synchronous behavior
process.on('uncaughtRejection', () => {
  console.log(`UnCaughtRejection is detected, shutting down server....`);
  process.exit(1);
});

```
> ### App
``` javascript
import cors from 'cors';
import express, { Application, Request, Response } from 'express';
import router from './app/routes';
import globalError from './app/middlewares/globalError';
import cookieParser from 'cookie-parser';
const app: Application = express();
//parsers
app.use(express.json());
app.use(
  cors({
    origin: [''],
  }),
);
app.use(cookieParser());
app.use('/api/v1', router);
const getAController = (req: Request, res: Response) => {
  const a = 10;
  res.send(a);
};

app.get('/', getAController);
app.use(globalError);
app.use(notFound);
export default app;

```
## <font color="red">UTILS!</font> 
> ### For Response
``` javascript
import { Response } from 'express';
type Tdata<T> = {
  success: boolean;
  message: string;
  data: T;
};
export const sendResponse = <T>(res: Response, result: Tdata<T>) => {
  return res.status(200).json({
    success: result.success,
    message: result.message,
    data: result.data,
  });
};
```
  
>### CatchAsync()
```javascript
import { NextFunction, RequestHandler, Request, Response } from 'express';

export const catchAsync = (fn: RequestHandler) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch((err) => next(err));
  };
};

```

## Middlewares
> ### globalError
```javascript

import { ErrorRequestHandler, NextFunction, Request, Response } from 'express';
import httpStatus from 'http-status';
import { ZodError, ZodIssue } from 'zod';

const globalError: ErrorRequestHandler = (err, req, res, next) => {
  let statusCode = err.status || httpStatus.INTERNAL_SERVER_ERROR || 500;
  let message = err.message || 'Something went Wrong';
  // module 14
  let errorSource: TErrorSource = [
    {
      path: '',
      message: 'Something went wrong',
    },
  ];
  // module 14
  if (err instanceof ZodError) {
    const simplifiedError = handleZodError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSource = simplifiedError?.errorSource;
  } else if (err.name === 'ValidationError') {
    const simplifiedError = handleValidationError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSource = simplifiedError?.errorSource;
  } else if (err.name === 'castError') {
    const simplifiedError = handleCastError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSource = simplifiedError?.errorSource;
  } else if (err.code === 11000) {
    const simplifiedError = handleDuplicateError(err);
    statusCode = simplifiedError?.statusCode;
    message = simplifiedError?.message;
    errorSource = simplifiedError?.errorSource;
  } else if (err instanceof AppError) {
    statusCode = err?.statusCode;
    message = err.message;
    errorSource = [
      {
        path: '',
        message: err?.message,
      },
    ];
  } else if (err instanceof Error) {
    message = err.message;
    errorSource = [
      {
        path: '',
        message: err?.message,
      },
    ];
  }
  return res.status(statusCode).json({
    success: false,
    message: err || message,
    errorSource,
    stack: config.node_env == 'development' ? err.stack : null,
  });
};
export default globalError;
/*
  \  pattern | 
  success
  message
  errorSources:[
    path:""
,
    message:''
  ]
  stack
*/
```
> ### notFound
```javascript
import { Request, Response } from 'express';
const notFound = (req: Request, res: Response) => {
  return res.status(404).json({
    success: false,
    message: 'API not found',
  });
};
export default notFound;
```
> ### validations
``` javascript
import { NextFunction, Request, Response } from 'express';
import { AnyZodObject } from 'zod';
export const validation = (schema: AnyZodObject) => {
  return catchAsync(async (req: Request, res: Response, next: NextFunction) => {
    await schema.parseAsync(req.body);
    next();
  });
};
```
## Errors
> ### AppError
```javascript
class AppError extends Error {
  public statusCode: number;
  constructor(statusCode: number, message: string, stack = '') {
    super(message);
    this.statusCode = statusCode;

    if (stack) {
      this.stack = stack;
    } else {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}
export default AppError;

```
> ### handleCastError
```javascript
import mongoose from 'mongoose';
import { TErrorSource, TGenericErrorResponse } from '../interface/error';
export const handleCastError = (
  err: mongoose.Error.CastError,
): TGenericErrorResponse => {
  const statusCode = 400;
  const errorSource: TErrorSource = [{ path: err.path, message: err.message }];
  return {
    statusCode,
    message: 'Invalid Id',
    errorSource,
  };
};
```
>### DuplicteError
```javascript
import { TErrorSource, TGenericErrorResponse } from '../interface/error';
export const handleDuplicateError = (err: any): TGenericErrorResponse => {
  const statusCode = 400;
  const match = err.message.match(/"([^"]*)"/);
  const extracted_msg = match && match[1];
  const errorSource: TErrorSource = [
    { path: '', message: ` ${extracted_msg} is already exists ` },
  ];
  return {
    statusCode,
    message: 'Duplicated Error',
    errorSource,
  };
};
```
>### ValidationError
```javascript
import mongoose from 'mongoose';
import { TErrorSource, TGenericErrorResponse } from '../interface/error';

export const handleValidationError = (
  err: mongoose.Error.ValidationError,
): TGenericErrorResponse => {
  const errorSource: TErrorSource = Object.values(err.errors).map((val) => {
    return {
      path: val.name,
      message: val.message,
    };
  });
  const statusCode = 400;
  return {
    statusCode,
    message: 'validation Error',
    errorSource,
  };
};

```
> ### Zod validation
```javascript
import { ZodError, ZodIssue } from 'zod';
import { TGenericErrorResponse } from '../interface/error';
// module 14
export const handleZodError = (err: ZodError): TGenericErrorResponse => {
  let statusCode = 400;
  let errorSource = err.issues.map((issue: ZodIssue) => {
    return {
      path: issue?.path[issue.path.length - 1],
      message: issue.message,
    };
  });
  return {
    statusCode,
    message: 'Zod Validation Error',
    errorSource,
  };
};
```

> ## Routes
``` javascript

import { Router } from 'express';
// import routes needs to be important

const router = Router();
const moduleRoutes = [
  {
    path: '/user',
    router: userRoute,
  },
  {
    path: '/student',
    router: StudentRoutes,
  },
  {
    path: '/academic-semester',
    router: academicSemester,
  },
  {
    path: '/academic-faculty',
    router: academicFacultyRoute,
  },
  {
    path: '/academic-department',
    router: academicDepartmentRoute,
  },
  {
    path: '/courses',
    router: courseRoute,
  },
  {
    path: '/semester-register',
    router: semesterRegisterRoute,
  },
  {
    path: '/auth',
    router: authRoutes,
  },
];
moduleRoutes.forEach((route) => {
  router.use(route.path, route.router);
});
export default router;
```
## QueryBuilder
``` javascript
import { FilterQuery, Query } from 'mongoose';
class QueryBuilder<T> {
  // modelQuery  =  all types of models;
  public modelQuery: Query<T[], T>;
  public query: Record<string, unknown>;
  constructor(modelQuery: Query<T[], T>, query: Record<string, unknown>) {
    this.modelQuery = modelQuery;
    this.query = query;
  }
  search(searchFields: string[]) {
    console.log(searchFields);
    const searchTerm = this?.query?.searchTerm;
    if (this?.query?.searchTerm) {
      this.modelQuery = this.modelQuery.find({
        $or: searchFields.map(
          (field) =>
            ({
              [field]: { $regex: searchTerm, $options: 'i' },
            }) as FilterQuery<T>,
        ),
      });
    }
    return this;
  }
  filter() {
    const queryObj = { ...this.query };
    //filtering
    const excluding = ['searchTerm', 'sort', 'limit', 'page', 'fields'];
    excluding.forEach((val) => delete queryObj[val]);
    this.modelQuery = this.modelQuery.find(queryObj as FilterQuery<T>);
    return this;
  }
  sort() {
    const sort =
      this?.query?.sort ||
      '-createdAt' ||
      (this?.query?.sort as string).split(',').join(' ');
    this.modelQuery = this?.modelQuery?.sort(sort as string);
    return this;
  }
  paginate() {
    const limit = Number(this?.query?.limit) || 1;
    const page = Number(this?.query?.page) || 1;
    const skip = (page - 1) * limit;
    this.modelQuery = this.modelQuery.skip(skip).limit(limit);
    return this;
  }
  fields() {
    const fields =
      this.query && this.query.fields && typeof this.query.fields === 'string'
        ? this.query.fields.split(',').join(' ')
        : '-__v' || null;

    this.modelQuery = this.modelQuery.select(fields);
    return this;
  }
}
export default QueryBuilder;

```

