# Application Architecture 

## Folder Structure

The application’s folder structure is modeled, prioritizing streamlined code organization, reduced redundancy, and a clearly defined, easily navigable project architecture.
```
simtestlab-ems/
├── src/                      
    ├──   app    
        ├── layout.tsx            # Root layout (providers, theme, auth)
        ├── page.tsx              # Landing/redirect page
        ├── action.ts             # Server actions (Next.js)
    ├── configs
        ├── IconDictionary.tsx
        ├── 
    ├── layouts
        ├── Authentication.tsx
        ├── layout.tsx
    ├──
        ├── utils.tsx
    ├──   modules
        ├── Auth 
            ├── component
            ├── page
            ├── store
            ├── hooks
            ├── utils
        ├── Live status
            ├── component
            ├── page
            ├── store
            ├── hooks
            ├── utils
        ├── Map
            ├── component
            ├── widgets
            ├── page
            ├── store
            ├── hooks
            ├── utils
        ├── Energy Flow
            ├──component
            ├── page
            ├── store
            ├── hooks
            ├── utils
        ├── Alerts
            ├── component
            ├── page
            ├── store
            ├── hooks
            ├── utils
        ├── Analytics
            ├── component
            ├── page
            ├── store
            ├── hooks
            ├── utils
    ├──   components
        ├── Header.tsx
        ├──Footer.tsx
    ├── store
        ├── alertstore.ts
        ├──themestore.ts
    ├── styles
        ├── global.css
        ├── index.ts
    ├── theme
        ├──provider.tsx
└── 