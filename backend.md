
Backend API Design and Security Practices in Node.js with Express:


1. **Authentication Route**: I'd create an `/auth/login` route that accepts email and password in the request body and validate using zod.
```javascript
import { z } from 'zod';
import rateLimit from 'express-rate-limit';

// Zod Schema
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: 'Too many login attempts from this IP, please try again after 15 minutes'
});

//function for response schema
function createResponse({ status, message, data = null, error = null }) {
  return {
    status: status === "success" ? "success" : "failed",
    message,
    data,
    error
  };
}

app.post('/auth/login', loginLimiter, async (req, res) => {
  try {
    //  Validating the request payload using Zod
    const { email, password } = await loginSchema.parseAsync(req.body);

    //  Querying the DB to find the User using their email address
    const user = await User.findOne({ email });
    if (!user) return res.status(401).json({ error: 'Invalid credentials' });

    const isPasswordMatch = await bcrypt.compare(password, user.password);
    if (!isPasswordMatch) return res.status(401).json({ error: 'Invalid credentials' });

    //  Generate a JWT token
    const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: '24h' });

   //Structure for all responses {status: "success" | "failed", message: string, data: any, error: any}
    res.json({
  status: "success",
  message: "Token generated successfully!",
  error: null,
  data: token
 });
  } catch (error) {
  
    if (error instanceof z.ZodError) {
      return res.status(400).json({ error: error.issues });
    }
    // Handle other errors that are nt zod
    res.status(500).json({
status: "success",
  message: "Token generated successfully!",
  error: 'Internal server error',
  data: null
 });
  }
});
```

2. **Authorization Middleware function to protect sensitive routes.**
```javascript
const authorize = async (req, res, next) => {
  try {
    // 1. Extract the token from the request headers
    const token = req.headers && req.headers.authorization && req.headers.authorization.split(' ')[1];

    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = await User.findById(decoded.userId);

    // 3. Pass control to the next middleware
    next();
  } catch (error) {
    res.status(401).json({ error: 'Unauthorized' });
  }
};
```

3. **Secure JWT**: 
   - Set the JWT `expiresIn` option to a reasonable duration (e.g., 24 hours).
   - Store the JWT secret in an environment variable and use double encryption techniques like HMAC to further secure the token.
   - Implement token revocation by storing revoked tokens in a blacklist or by including a `jti` (JWT ID) claim in the token and maintaining a list of valid JTIs.

4. **Use bcrypt to hash passwords before storing them in the database.**: 
```javascript
const hashedPassword = await bcrypt.hash(password, 10);
```

5. **PApply the `authorize` middleware to routes that require authentication.**: 
```javascript
app.get('/profile', authorize, (req, res) => {
  res.json(req.user.profile);
});
```

          
