# Architecture with atomic design

## Why this doc exists

I have not found a good explanation of how to combine atomic design with react components. Consider this document as the best practices that work for me. Feel free to give feedback with fresh ideas and improvements.

## Atom

- Should contain `className` property for the ability to style atom from outside (page, molecule, another place)

```jsx
// SomePage.tsx component
import Avatar from '@components/atoms/Avatar';

const Page = ({ ...props }) => {
  // ...logic
	

  return (
	  <Avatar {...props} />
  );
};

export default Page;

const AvatarStyled = styled(Avatar)`
  margin: 15px 20px;
  // ...another custom styles
`;

// Avatar.tsx atom component
const Avatar = ({ className, ...restProps }) => {
  return (
    // gives ability to do AvatarStyled approach
    // https://styled-components.com/docs/advanced#styling-normal-react-components
    <div className={className}></div>
}
```

- Sometimes atoms can be treated as molecules. The frontend team (or team lead) should resolve such issues. Avatar is a circle with `firstName` and `lastName` or image.  Split component more doesn't have sense. Button, Input, Checkbox are good examples of Atoms.
- If you use custom design the designer often provides such styling. [https://react-bootstrap.github.io/components/alerts](https://react-bootstrap.github.io/components/alerts) is a good example of pre-created atomic components. These components can be extended for custom needs.
- Use general naming. `Avatar`, `Button` are a good names. `EmployeeModalFooter` is a bad one (hard to reuse outside Employee page or ouside modal)
- If you need custom `UserButton` only once per project - create it inside `pages/UserPage` . There is no need to extract every component to atoms/molecules/organisms. Atomic design is about reusability of components.

```jsx
type Props = {
  className?: string;
  userStatus?: AvatarUserStatus;
  size?: AvatarSize;
  shape?: AvatarShape;
  src?: string;
  firstName?: string;
  lastName?: string;
};

const Avatar = ({
  className,
  userStatus,
  size = AvatarSize.standard,
  shape = AvatarShape.ellipse,
  src,
  firstName,
  lastName,
}: Props) => {
  const fullName = `${firstName ?? ''} ${lastName ?? ''}`;
  const userInitials = getUserInitials(firstName, lastName);

  return (
    <Root className={className} $size={size}>
      <ImageContainer shape={shape}>
        {src && <Image src={src} alt={fullName} />}

        {!src && (
          <AvatarPlaceholder $size={size}>
            <InitialsContainer>{userInitials}</InitialsContainer>
          </AvatarPlaceholder>
        )}
      </ImageContainer>
      {userStatus && <Status $status={userStatus} $size={size} />}
    </Root>
  );
};

export default Avatar;

const Root = styled.div<{ $size: AvatarSize }>`
  position: relative;

  width: ${(props) => props.$size}px;
  height: ${(props) => props.$size}px;
`;

// ... other styling
```

Example of `Avatar` atom component. 

## Molecule

- A composition of atoms (or atom + smth else)
- Often these are components that are used together in many places. (`Input` atom with close image → `SearchInput` molecule

```jsx
import Checkbox, { Label } from '../../_atoms/Checkbox';

export type Props = {
  label: string;
  titleLabel?: string;
  disabled?: boolean;
  className?: string;
  onChange?: (event: React.ChangeEvent<HTMLInputElement>) => void;
  checked?: boolean;
  hasCheckbox?: boolean;
};

const Option: React.FC<Props> = ({
  label,
  titleLabel = '',
  disabled = false,
  checked = false,
  hasCheckbox = false,
  onChange = () => {},
  className = '',
}) => (
  <Wrapper
    className={className}
    disabled={disabled}
    checked={checked}
    title={titleLabel}
  >
    {hasCheckbox && (
      <CheckboxStyled
        disabled={disabled}
        checked={checked}
        onChange={onChange}
        label={label}
      />
    )}
    {!hasCheckbox && (
      <LabelStyled
        disabled={disabled}
        checked={checked}
        onClick={!disabled ? (onChange as any) : () => {}}
      >
        {label}
      </LabelStyled>
    )}
  </Wrapper>
);

export default Option;

const Wrapper = styled`
 // some styling
`;

const LabelStyled = styled(Label)`
  // some styling
`;

const CheckboxStyled = styled(Checkbox)`
	padding: ${theme.spacing.gap12} ${theme.spacing.gap16};
`

// ... other styling
```

Example of molecule `Option` component.

## Organism

- Composition of atom(s) and (or) molecules(s), sometimes with its own logic.
- The Dropdown that contains `SearchField` for search, `Option` with `Checkbox` es. There is a logic and types for handing `Checkbox` selection, clearing all items.

```jsx
import Button, { ButtonVariants } from '../../_atoms/Button';
import ModalContainer from '../../_atoms/ModalContainer';
import Option, { Props as OptionProps } from '../../_molecules/Option';
import SearchField from '../../_molecules/SearchField';

const checkOption = R.assoc('checked', true);
const uncheckOption = R.assoc('checked', false);
const hasAllChecked = R.all(R.propEq('checked', true));
const hasAllUnchecked = R.all(R.propEq('checked', false));

export type DropdownOption = OptionProps & { id: number | string };

export type Props = {
  options: DropdownOption[];
  onChange: (options: DropdownOption[]) => void;
  searchPlaceholder?: string;
  className?: string;
};

const DropdownListOneGroup: React.FC<Props> = ({
  options,
  onChange,
  searchPlaceholder = 'Find options...',
  className = '',
}) => {
  const [searchQuery, setSearchQuery] = useState('');

  const handleOptionChange = (option: DropdownOption) => {
    const newOption = R.assoc('checked', !option.checked, option);
    const newOptions = R.map(
      (opt) => (opt.id === option.id ? newOption : opt),
      options,
    );

    onChange(newOptions);
  };

  const handleClearSelectedOptions = () => {
    onChange(R.map(uncheckOption, options));
  };

  const handleSelectAllOptions = () => {
    onChange(R.map(checkOption, options));
  };

  const searchQueryMatch: (list: DropdownOption) => boolean = R.pipe(
    R.prop('label'),
    R.toLower,
    R.includes(R.toLower(searchQuery)),
  );

  const filteredBySearchQuery = options.filter(searchQueryMatch);
  const checkedOptions = filteredBySearchQuery.filter(
    R.propEq<any>('checked', true),
  );
  const uncheckedOptions = filteredBySearchQuery.filter(
    R.propEq<any>('checked', false),
  );

  const hasAllUncheckedOptions = hasAllUnchecked(filteredBySearchQuery);
  const hasAllCheckedOptions = hasAllChecked(filteredBySearchQuery);

  return (
    <ModalContainer className={className}>
      <SearchField
        value={searchQuery}
        onChange={setSearchQuery}
        placeholder={searchPlaceholder}
      />

      <List>
        {hasAllUncheckedOptions && (
          <ButtonStyled
            variant={ButtonVariants.tertiary}
            onClick={handleSelectAllOptions}
          >
            Select all
          </ButtonStyled>
        )}
        {hasAllCheckedOptions && (
          <ButtonStyled
            variant={ButtonVariants.tertiary}
            onClick={handleClearSelectedOptions}
          >
            Deselect all
          </ButtonStyled>
        )}

        {!hasAllUncheckedOptions && !hasAllCheckedOptions && (
          <ButtonStyled
            variant={ButtonVariants.tertiary}
            onClick={handleClearSelectedOptions}
          >
            Clear selected
          </ButtonStyled>
        )}
        {checkedOptions.map((option) => (
          <Option
            key={option.id}
            onChange={() => handleOptionChange(option)}
            label={option.label}
            disabled={option.disabled}
            checked={option.checked}
            hasCheckbox={option.hasCheckbox}
          />
        ))}

        {!hasAllUncheckedOptions && !hasAllCheckedOptions && (
          <ButtonStyled
            variant={ButtonVariants.tertiary}
            onClick={handleSelectAllOptions}
          >
            Select all
          </ButtonStyled>
        )}
        {uncheckedOptions.map((option) => (
          <Option
            key={option.id}
            onChange={() => handleOptionChange(option)}
            label={option.label}
            disabled={option.disabled}
            checked={option.checked}
            hasCheckbox={option.hasCheckbox}
          />
        ))}
      </List>
    </ModalContainer>
  );
};

export default DropdownListOneGroup;

const ButtonStyled = styled(Button)`
  color: var(--ui-text);
  font-size: 12px;
  align-self: flex-end;
`;

const List = styled.div.attrs({
  className: 'custom-scrollbar',
})`
  overflow: auto;
  margin: var(--gap16) 0;
  max-height: 256px;
`;
```

Example of `Dropdown` organism component.

# Template

- Optional. Not every page should have it.
- Used when exists common placing of components on the page.

```jsx
type Props = {
  filters: JSX.Element;
  content: JSX.Element;
  title?: string;
};

const Template = ({ filters, content, title = 'User permissions' }: Props) => (
  <Root>
    <Header>
      <HeaderContainer>
        <PageTitle>{title}</PageTitle>
      </HeaderContainer>
    </Header>
    <FullWidthDivider />
    <FiltersContainer>{filters}</FiltersContainer>
    <TableContainer>{content}</TableContainer>
  </Root>
);

export default Template;

const Root = styled.div`
  position: relative;
  height: 100vh;
  background: var(--background-main);
  display: grid;
  grid-template-columns: auto;
  grid-template-rows: 64px 1px 122px calc(100% - 203px);
  grid-template-areas:
    'header'
    'divider'
    'filters'
    'table';
`;

const Header = styled.div`
  display: flex;
  width: 100%;
  background: var(--background-main);
  grid-area: header;
`;

const FiltersContainer = styled.div`
  width: 100%;
  padding: var(--gap24) var(--gap56);
  grid-area: filters;
`;

// ...another styling
```

Example of `Template` template component.

# Page

- Often contains requests, specific logic, handlers

```jsx
type Props = {
  userProfile: UserProfileType | null;
  isReadonlyPermission: boolean;
  searchQuery: string;
  isNewModalOpened: boolean;
  setIsNewModalOpened: Dispatch<SetStateAction<boolean>>;
};

const UserPage: React.FC<Props> = ({
  userProfile,
  isReadonlyPermission,
  searchQuery,
  isNewModalOpened,
  setIsNewModalOpened,
}) => {
  const { setNotification } = useContext(NotificationContext);

  const { users, usersData, createUser, refetchUsers } = useUsers({
    searchQuery,
  });

  const { userRoleData } = useUserRole();
  const { userGroupsData } = useUserGroups();

  const handleCreate = (options: CreateUserType) =>
    createUser.mutateAsync(options, {
      onSuccess: (data) => {
        setIsNewModalOpened(false);
        setNotification({
          status: NotificationTypes.SUCCESS,
          text: `User ${data.firstName} ${data.lastName} was created`,
        });

        refetchUsers();
      },
    });

  const loadingStatus = !userProfile || users.isLoading;
  const noResultsStatus = // ...
  const viewingStatus = // ...

  const notNullUserProfile = // ...;
  const notNullUserData = // ...;

  return (
    <>
      {loadingStatus && (
        <PlaceholderContainer>
          <Spinner size={SpinnerSizes.Huge} />
        </PlaceholderContainer>
      )}

      {noResultsStatus && <NoData header="No results found" message="" />}

      {viewingStatus && (
        <UserTable
          data={notNullUserData}
          userProfile={notNullUserProfile}
          isReadonlyPermission={isReadonlyPermission}
          roles={userRoleData}
          groups={userGroupsData}
        />
      )}

      {isNewModalOpened && (
        <UserModal
          onClose={() => setIsNewModalOpened(false)}
          onSubmit={handleCreate}
          userProfile={notNullUserProfile}
          rolesList={userRoleData}
          groupsList={userGroupsData}
          isActive={isNewModalOpened}
          isReadonlyPermission={isReadonlyPermission}
        />
      )}
    </>
  );
};

export default UserPage; 
```

Example of the `User` page component.

## References

1. [https://bradfrost.com/blog/post/atomic-web-design/](https://bradfrost.com/blog/post/atomic-web-design/)
2. [https://habr.com/ru/post/249223/](https://habr.com/ru/post/249223/)
3. [https://www.youtube.com/watch?v=fqULJBBEVQE&ab_channel=GoogleDevelopers](https://www.youtube.com/watch?v=fqULJBBEVQE&ab_channel=GoogleDevelopers)